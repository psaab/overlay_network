# Technical Design Specification: XDP-Prox (High-Performance Userspace Stack)

## 1. Introduction
This document defines the modular architecture for `XDP-Prox`, a userspace networking stack. It provides a secure userspace mediation layer between VM virtio traffic and physical NIC queues.

## 2. High-Level Architecture
- **Data Plane:** High-performance packet processing loop running on isolated CPU cores.
- **Control Plane:** Manages rules, monitoring, and stack configuration via epoch/RCU snapshots.

### 2.1 Memory Model: Logical Isolation & UMEM
To achieve high-performance forwarding, the stack utilizes a contiguous UMEM region.
- **Per-NUMA UMEM Pools:** One UMEM pool per NUMA node; on a single-socket host this collapses to one pool. Each NIC RX/TX queue is bound to the UMEM on the queue's NUMA node so that DMA descriptors never cross sockets on the fast path.
- **Logical Isolation:** Slicing UMEM per-VM provides logical isolation only — it is not a memory-protection boundary. Strict descriptor bounds checking is enforced. Hard isolation across trust zones requires a separate UMEM (and likely a separate process); see §7.
- **Zero-Copy vs Single-Copy:** vhost-user descriptors and AF_XDP descriptors reference different memory domains. The baseline implementation copies packet bytes from guest buffers into AF_XDP UMEM on egress and from AF_XDP UMEM into guest buffers on ingress. Descriptor passing is allowed only if a future architecture makes guest packet buffers valid AF_XDP UMEM frames and satisfies DMA isolation requirements; out of scope for v1.

### 2.2 Offload Contract
- Virtio-net features (TSO/GSO/UFO/GRO/LRO, mergeable rxbuf, guest checksum-partial) are disabled in v1; re-enabled later behind correctness gates.
- AF_XDP single-buffer mode in v1; max MTU 1500; UMEM chunk size 2048 bytes with 256-byte headroom (multi-buffer / jumbo deferred).
- **L3 protocol scope (v1):** IPv6 only. Non-IPv6 EtherTypes (including IPv4 0x0800 and ARP 0x0806) are dropped at parse stage with reason `non_ipv6_dropped`. IPv4-via-encap is a future revision.
- **IPv6 forwarding rewrites:** The dataplane decrements the IPv6 hop-limit on every forwarded packet. IPv6 has no L3 header checksum, and hop-limit is not part of any pseudo-header, so hop-limit decrement does not require any L4 checksum recompute.
- **No L3 NAT in v1:** Source and destination IPv6 addresses are not modified on the v1 hot path. TCP/UDP pseudo-header recompute infrastructure is preserved in code for the future encapsulation/IPv4 path but is unused on the v1 forwarding fast path.
- **TCP/UDP checksums:** Pass-through on forwarded flows. IPv6 UDP requires non-zero checksums; packets with zero UDP checksums are dropped.
- **Ethernet FCS:** Computed by the NIC on TX and stripped on RX — not present in host memory.

## 3. Module Breakdown

### 3.1 Module: I/O Engine & Queue Topology
- **Topology:** One XSK per NIC queue, one dataplane thread per queue group, one UMEM pool per NUMA node.
- **XDP Program Pseudocode:**
  ```c
  index = ctx->rx_queue_index;
  if (bpf_map_lookup_elem(&xsks_map, &index))
      return bpf_redirect_map(&xsks_map, index, 0);
  return XDP_DROP; // Fail-closed
  ```
- **TX completion drain:** Drained per RX poll iteration with a bounded per-iteration cap; UMEM frames are recycled to the FILL ring immediately on completion to prevent UMEM starvation under sustained load.
- **Wakeup model:** `XDP_USE_NEED_WAKEUP` enabled; busy-poll on dataplane cores. Syscall wakeup paths are a fallback for low-rate queues.
- **BPF map specs:** `xsks_map` is `BPF_MAP_TYPE_XSKMAP`, sized to NIC queue count, pinned at `/sys/fs/bpf/xdp-prox/xsks_map`. XDP program type is `BPF_PROG_TYPE_XDP`, attached to the physical NIC in native (driver) mode via `XDP_FLAGS_DRV_MODE`. Required capabilities (`CAP_BPF`, `CAP_NET_ADMIN`) are dropped after attach; `CAP_IPC_LOCK` is retained for UMEM mlock.
- **Detach policy:** The XDP program remains attached on dataplane exit so traffic continues to fail-closed-DROP until explicitly detached by an operator command.

### 3.2 Module: Flow Classifier & Conntrack
- **Multi-Tenant Awareness:** Flow key includes tenant/VM-ID, ingress interface, direction, L3 protocol, 5-tuple, and conntrack zone.
- **State & Limits:** Full TCP/UDP state tracking. Per-core timer wheels for timeouts. Per-tenant memory quotas and SYN flood eviction.

### 3.3 Module: VM Steering & Anti-Spoofing
- **Identity Verification:** Populate the Steering Table from control-plane VM identity. Observed source MACs may be logged, but untrusted VM traffic must not authoritatively update forwarding ownership. Enforce provisioned MAC, IPv6 (one or more global IPv6 bindings per VM; v1 is GUA-only), and VLAN bindings before conntrack.
- **Link-local handling:** VMs may use any RFC 4291-compliant link-local source (in `fe80::/64`) for on-link ND traffic (NS/NA/RS to or from any on-link host, including the gateway and other VMs), but must not use link-local as the source of forwarded (off-link) traffic.
- **ND/RA guard rules** (per RFC 4861/4862; valid VM-sourced ND messages):
  - **NS (type 135):** allowed if either (a) source = VM's provisioned IPv6 or link-local AND target is a valid on-link address (gateway, another on-link host, or one of the VM's own addresses); or (b) source = unspecified `::` AND IP destination = the solicited-node multicast address (`ff02::1:ffXX:XXXX`) corresponding to the Target Address AND Target Address = one of the VM's own addresses, including its link-local address (Duplicate Address Detection per RFC 4862 §5.4, which mandates DAD on all unicast addresses including link-local; destination check per RFC 4861 §7.1.1).
  - **NA (type 136):** allowed if source = any of the VM's provisioned addresses or its link-local address, AND the Target Address field in the NA payload is one of the VM's own addresses (provisioned or link-local). The R (Router) bit MUST be zero (VMs are not routers); NAs with R=1 are dropped.
  - **RA (type 134):** dropped unconditionally (RA guard, RFC 6105). VMs are never legitimate sources of router advertisements.
  - **RS (type 133):** allowed from any of the VM's provisioned addresses, its link-local address, or `::` (initial bootstrap before any address is assigned); the dataplane responds with an RA according to its RA-emission policy.
  - **Redirect (type 137):** dropped unconditionally from VMs (only routers send Redirects).
  - **Hop-limit check (RFC 4861 §11.2):** all received ND messages must have IPv6 hop-limit = 255; otherwise dropped (`nd_hop_limit_invalid`). This prevents off-link spoofing of ND.
  - **Fragmentation (RFC 6980):** ND messages must not be fragmented; fragmented ND is dropped (`nd_fragmented_dropped`).
  - **ICMPv6 sanity:** code, length, and checksum are validated. Zero-length options drop with `nd_malformed` (would loop the parser). Unknown ND option types are silently ignored per RFC 4861 §4.6 — message processing continues with the recognized options.
  - **Link-layer-address option validation (RFC 4861 §4.6.1):** for the VM-sourced ND messages that survive the rules above (NS, RS, NA), any SLLAO (in NS/RS) or TLLAO (in NA) must equal the VM's provisioned MAC. Mismatched options drop with `nd_lladdr_mismatch` to prevent neighbor-cache poisoning of the VM's own entries with a forged link-layer address. (RA and Redirect are already dropped unconditionally above; their option fields are never reached.)
  - **`::`-source SLLAO prohibition (RFC 4861 §4.1, §7.1.1):** NS or RS with source `::` MUST NOT carry an SLLAO; presence of an SLLAO on such a packet is a protocol violation. Drop with `nd_unspecified_with_sllao` before the MAC-match step.
- **Rate Limiting:** Applied before shared TX queues. Limits on bytes and packets-per-second (pps) to avoid CPU exhaustion. ICMPv6 generation by the dataplane is rate-limited per (tenant, error-type).

### 3.4 Module: Security Engine & Parsers
- **Strict Parsing:** Bounds-check all headers. Parse IPv6 extension header chains with a bounded depth (default: 4 extension headers). Drop malformed or ambiguous packets before conntrack. Non-IPv6 EtherTypes are dropped at L2 parse.
- **DPI Constraints:** DPI runs on a separate CPU budget. Regex-engine scratch regions (Vectorscan/Hyperscan-compatible) are allocated per dataplane thread (no fast-path sharing).

### 3.5 Module: IPv6 Routing & Neighbor Discovery
- **FIB:** Longest-prefix-match IPv6 FIB. Populated from control-plane configuration; physical-side route learning (RA, BGP, OSPFv3) is a v1 open question (DESIGN.md §7).
- **Hop-limit:** All forwarded packets have their hop-limit decremented. Packets arriving with hop-limit ≤ 1 are dropped, and an ICMPv6 Time Exceeded (type 3, code 0) is emitted to the source, rate-limited per (tenant, source).
- **VM-facing Neighbor Discovery:** The dataplane responds to NS for the gateway's link-local and global addresses on each VM-facing link. NS for VM-owned addresses are answered via proxy-ND from the provisioned identity table — the dataplane never relies on observing VM source addresses to learn ownership. ND messages must satisfy the validation rules in §3.3.
- **Physical-side Neighbor Discovery:** On the physical-facing interface, the dataplane participates as an ordinary IPv6 host/router. It performs NS to resolve upstream router(s)' link-layer addresses, processes inbound NA, and (depending on the routing-participation choice in DESIGN.md §7) may consume RAs from the upstream to populate the default route. Inbound ND from the physical side is also subject to RFC 4861 §11.2 hop-limit=255 and RFC 6980 no-fragmentation checks; the upstream link is otherwise trusted (no ND-guard).
- **Duplicate Address Detection (RFC 4862 §5.4):** The dataplane performs DAD on its own link-local and global addresses at startup before declaring an interface up. VMs perform DAD on their own assigned addresses; the dataplane allows their `::`-sourced NS per the §3.3 rules and does not respond — DAD is host-to-host on the link. Care is taken that proxy-ND for a VM-owned address does NOT respond to a DAD probe sourced from `::` for that same address (which would falsely indicate a duplicate).
- **Router Advertisements:** RA emission policy (interval, prefix info, M/O flags driving SLAAC vs DHCPv6, RDNSS) is configured by the control plane; default is a v1 open question (DESIGN.md §7).
- **ICMPv6 Redirect (RFC 4861 §8):** The dataplane neither generates Redirects (it is the only first-hop router on each VM-facing link) nor accepts them from VMs (drop with `redirect_dropped`).
- **ICMPv6 Errors generated:** Time Exceeded, Packet Too Big (PTB), Parameter Problem, and Destination Unreachable. Per-(tenant, error-type) token-bucket rate limits prevent amplification. Source-address selection for these messages follows RFC 6724 (open question DESIGN.md §7 pending choice of preferred source on multi-bound links).
- **Path MTU:** v1 has uniform MTU 1500 across VM and physical sides; PTB generation is dormant unless future MTU heterogeneity is introduced.
- **Multicast:** Link-local IPv6 multicast (`ff02::/16`) required for ND (solicited-node `ff02::1:ffXX:XXXX`, all-nodes `ff02::1`, all-routers `ff02::2`) is handled locally by the dataplane. Other multicast scopes are not forwarded in v1; MLD (RFC 3810) is therefore functionally N/A for forwarding, though the dataplane may still emit MLDv2 reports for groups it joins itself.

## 4. Packet Processing Pipeline

1. **Ingress:** `XSK_RX` batch fetch.
2. **Parse & Anti-Spoof:** Validate L2/L3 bounds and VM identity.
3. **Lookup:** Flow Classifier check.
4. **Slow Path:** ACLs, metadata DPI, Conntrack creation.
5. **Forward Action:** IPv6 FIB lookup, hop-limit decrement, per-tenant rate limit. No L3 address rewrite in v1 (no NAT). Flows tagged `forward-to-proxy` are queued for the deferred proxy delivery path (see DESIGN.md §2); in v1 they are dropped at this stage with reason `proxy_deferred`.
6. **Egress:** TX backpressure handling (tail-drop if full).

## 5. Performance Targets & Budgets
- **Wire Speed Envelope:** 10 GbE target (14.88 Mpps at 64B frames).
- **Cache & NUMA:** UMEM and rings allocated on same NUMA node as NIC. Cacheline-align hot counters.
- **CPU Features:** SSE4.2/AVX2 baseline for DPI.

## 6. Testing & Observability
- **Testing Layers:** Fuzzing parsers (cargo-fuzz), pcap replays, chaos testing for proxy outage and ring exhaustion.
- **Metrics:** RX/TX ring occupancy, fill/completion starvation, drop reason enumerations, per-path p99 latency, LLC misses.
- **Rust Safety:** Unsafe boundaries are explicitly audited. Packet buffers wrapped in safe types holding length and headroom.

## 7. Security Considerations
- **Fail-Closed:** Default drop on missing config or unparsed headers. Control-plane crash retains the last-published policy snapshot (see DESIGN.md §5).
- **L3 Protocol Filter:** Non-IPv6 traffic from VMs is dropped at parse stage (`non_ipv6_dropped`). No IPv4 or ARP forwarding path exists in v1.
- **No NAT Surface:** With no NAT/DNAT in the forwarding plane, the conntrack table holds flow state but no translation tuples; the entire class of NAT-related vulnerabilities (port-pool exhaustion, hairpin loops, NAT slipstreaming) is out of scope.
- **Epoch Config Reload:** Transactional config updates with RCU reclamation.