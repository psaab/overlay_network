# Technical Design Specification: XDP-Prox (High-Performance Userspace Stack)

## 1. Introduction
This document defines the modular architecture for `XDP-Prox`, a userspace networking stack. It provides a secure userspace mediation layer between VM virtio traffic and physical NIC queues.

## 2. High-Level Architecture
- **Data Plane:** High-performance packet processing loop running on isolated CPU cores.
- **Control Plane:** Manages rules, monitoring, and stack configuration via epoch/RCU snapshots.

### 2.1 Memory Model: Logical Isolation & UMEM
To achieve high-performance forwarding, the stack utilizes a contiguous UMEM region.
- **Global Pool:** Shared for physical NIC I/O.
- **Logical Isolation:** Slicing UMEM per-VM provides logical isolation. Strict descriptor bounds checking is enforced.
- **Zero-Copy vs Single-Copy:** vhost-user descriptors and AF_XDP descriptors reference different memory domains. The baseline implementation copies packet bytes from guest buffers into AF_XDP UMEM on egress and from AF_XDP UMEM into guest buffers on ingress. Descriptor passing is allowed only if the selected architecture makes guest packet buffers valid AF_XDP UMEM frames and satisfies DMA isolation requirements.

### 2.2 Offload Contract
- Virtio-net features (TSO/GSO/GRO) are disabled in v1. 
- Header rewrites recalculate Ethernet FCS, IPv4 checksums, and TCP/UDP pseudo-headers explicitly.

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
5. **Proxy/NAT:** Rewrite headers; push to proxy namespace if required.
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
- **Fail-Closed:** Default drop on missing config, dead proxy, or unparsed headers.
- **Epoch Config Reload:** Transactional config updates with RCU reclamation.