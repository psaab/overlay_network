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
- Header rewrites recalculate IPv4 header checksums and TCP/UDP checksums (including pseudo-header). Ethernet FCS is computed by the NIC on TX and stripped on RX — it is not present in host memory and is therefore not a concern for the dataplane.
- IPv4 UDP checksum-zero is preserved on rewrite. IPv6 UDP requires non-zero checksums; the dataplane recomputes rather than passing through.

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
- **Identity Verification:** Populate the Steering Table from control-plane VM identity. Observed source MACs may be logged, but untrusted VM traffic must not authoritatively update forwarding ownership. Enforce provisioned MAC/IP/VLAN bindings before conntrack.
- **Rate Limiting:** Applied before shared TX queues. Limits on bytes and packets-per-second (pps) to avoid CPU exhaustion.

### 3.4 Module: Security Engine & Parsers
- **Strict Parsing:** Bounds-check all headers. Parse IPv4/IPv6 options with bounded depth. Drop malformed packets before conntrack.
- **DPI Constraints:** DPI runs on a separate CPU budget. Hyperscan scratch regions are allocated per dataplane thread (no fast-path sharing).

## 4. Packet Processing Pipeline

1. **Ingress:** `XSK_RX` batch fetch.
2. **Parse & Anti-Spoof:** Validate L2/L3 bounds and VM identity.
3. **Lookup:** Flow Classifier check.
4. **Slow Path:** ACLs, metadata DPI, Conntrack creation.
5. **Forward Action:** Apply L3 routing rewrites (NAT, if any) and per-tenant rate limits. Flows tagged `forward-to-proxy` are queued for the deferred proxy delivery path (see DESIGN.md §2); in v1 they are dropped at this stage with reason `proxy_deferred`.
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
- **Epoch Config Reload:** Transactional config updates with RCU reclamation.