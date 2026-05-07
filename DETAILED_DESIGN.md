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
- **Identity Verification:** Populate the Steering Table from control-plane VM identity. Observed source MACs may be logged, but untrusted VM traffic must not authoritatively update forwarding ownership. Enforce provisioned MAC, IPv6 (one or more global/ULA bindings per VM), and VLAN bindings before conntrack.
- **Link-local handling:** VMs may use any RFC 4291-compliant link-local source (`fe80::/64`) for ND traffic to the gateway, but must not use link-local as the source of forwarded (off-link) traffic.
- **ND/RA guard:** VM-sourced router advertisements (ICMPv6 type 134) are dropped unconditionally. VM-sourced NS/NA (types 135/136) are validated against the provisioned identity — the source IPv6 and target address must both belong to the VM.
- **Rate Limiting:** Applied before shared TX queues. Limits on bytes and packets-per-second (pps) to avoid CPU exhaustion. ICMPv6 generation by the dataplane is rate-limited per (tenant, error-type).

### 3.4 Module: Security Engine & Parsers
- **Strict Parsing:** Bounds-check all headers. Parse IPv6 extension header chains with a bounded depth (default: 4 extension headers). Drop malformed or ambiguous packets before conntrack. Non-IPv6 EtherTypes are dropped at L2 parse.
- **DPI Constraints:** DPI runs on a separate CPU budget. Regex-engine scratch regions (Vectorscan/Hyperscan-compatible) are allocated per dataplane thread (no fast-path sharing).

### 3.5 Module: IPv6 Routing & Neighbor Discovery
- **FIB:** Longest-prefix-match IPv6 FIB. Populated from control-plane configuration; physical-side route learning (RA, BGP, OSPFv3) is a v1 open question (DESIGN.md §7).
- **Hop-limit:** All forwarded packets have their hop-limit decremented. Packets arriving with hop-limit ≤ 1 are dropped, and an ICMPv6 Time Exceeded (type 3, code 0) is emitted to the source, rate-limited per (tenant, source).
- **Neighbor Discovery (gateway-side):** The dataplane responds to NS for the gateway's link-local and global addresses on each VM-facing link. NS for VM-owned addresses are answered via proxy-ND from the provisioned identity table — the dataplane never relies on observing VM source addresses to learn ownership.
- **Router Advertisements:** RA emission policy (interval, prefix info, M/O flags driving SLAAC vs DHCPv6) is configured by the control plane; default is a v1 open question (DESIGN.md §7).
- **ICMPv6 Errors:** Time Exceeded, Packet Too Big (PTB), Parameter Problem, and Destination Unreachable are generated by the dataplane on relevant conditions. Per-(tenant, error-type) token-bucket rate limits prevent ICMPv6 amplification.
- **Path MTU:** v1 has uniform MTU 1500 across VM and physical sides; PTB generation is dormant unless future MTU heterogeneity is introduced.
- **Multicast:** Link-local IPv6 multicast (`ff02::/16`) for ND is handled locally by the dataplane (solicited-node, all-nodes, all-routers). Other multicast scopes are dropped in v1.

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