# Userspace Networking Stack Architecture (AF_XDP)

## 1. Executive Summary
This document outlines the design and implementation strategy for a high-performance, userspace networking stack. The goal is to provide a secure userspace mediation layer between VM virtio traffic and physical NIC queues. The physical NIC path uses AF_XDP zero-copy where the driver supports it; the VM boundary is explicitly modeled as single-copy unless a separately specified shared-memory design is selected. The stack enforces deep packet inspection (DPI), wire-speed firewalling, and transparent proxy forwarding, ensuring the physical production network remains protected from untrusted VM traffic.

## 2. Architecture Overview

The system sits as a transparent, high-speed mediator between the VM's network interface and the physical uplink.

### Network Mode
The primary mode for v1 is a **Routed gateway with selective proxy**. The system owns the gateway MAC and IP for each VM-facing subnet. Flows are routed/NAT'd by default; selected flows (per ACL) are diverted to the proxy delivery path before egress.
- Gateway MAC and IPv4 address are owned by the system. Gateway IPv6 link-local (`fe80::/64`) and global IPv6 addresses are also assigned where IPv6 is enabled.
- ARP proxying and IPv6 ND for the gateway and for off-link destinations are explicitly managed by the dataplane (detailed plane is a v1 open question — see §7).
- DHCP plane (server vs relay vs neither) is a v1 open question — see §7. DHCP guard is dependent on this decision.

### Core Technologies
- **AF_XDP Sockets (XSK):** For raw, high-throughput packet I/O on netdev queues. Zero-copy is available only for supported native-XDP NIC drivers and only for buffers registered in AF_XDP UMEM. VM-facing TAP/veth and vhost-user paths have separate copy and feature-negotiation constraints.
- **eBPF/XDP Programs:** Minimal eBPF programs attached to the physical interface whose sole purpose is to redirect packets into the AF_XDP sockets (`XDP_REDIRECT`), or `XDP_DROP` if the socket/queue is unconfigured.
- **Rust (Recommended):** For memory safety, aggressive concurrency (lock-free structures), and high performance.

### VM-Facing Interface
We explicitly split support into two mutually exclusive modes (v1 will target vhost-user):
- **vhost-user backend mode:** High performance, requires implementing virtio-net backend semantics (split virtqueues, multiqueue, eventfd kick/calls). Virtio-net offloads (TSO/GSO/UFO/GRO/LRO, mergeable rxbuf, guest checksum-partial) are disabled in v1; re-enabled later behind correctness gates. Live migration and reconnect are out of scope for v1. Exact negotiated feature mask is a v1 open question — see §7.
- **TAP/veth mode (Fallback/Testing):** Simpler integration, copy-mode, lower peak performance.

### Proxy Delivery Path
For selected flows (per ACL), the dataplane forwards traffic to a downstream service that decides what to do with it. **The forwarding mechanism is deferred to a future design pass.** Candidate approaches (DNAT to a remote service, encapsulation, in-process termination, transparent SOCKS-ification, kernel re-entry) differ materially in copy budget, metadata conveyance, and protocol scope; the choice depends on what the downstream service expects to receive. v1 reserves an ACL action `forward-to-proxy` and the conntrack hooks needed to track such flows, but does not implement the forwarding path itself.

Independent of mechanism, when proxying is implemented:
- TLS interception is not assumed. DPI is limited to metadata observable in plaintext (SNI/ALPN/JA3-class).
- Default DROP policy for QUIC (UDP/443) to prevent TLS inspection bypass; UDP/QUIC on non-443 ports is a separate open question.

## 3. Key Components

### 3.1. I/O Engine
Manages Fill, Completion, RX, and TX rings. The physical side uses AF_XDP, while the VM side uses `vhost-user`. 

### 3.2. Fast-Path Connection Tracker (Conntrack)
A flow table tracking tenant/VM-ID, ingress interface, direction, L3 protocol, and normalized 5-tuple. It maintains full TCP state (SYN/ACK/FIN/RST) to prevent state exhaustion.

### 3.3. Firewall Rules Engine
Evaluates packets against ACLs. Enforces anti-spoofing before conntrack (validating source MAC, IP, VLAN against provisioned VM identity). IPv4/IPv6 reassembly runs upstream of anti-spoofing and conntrack so that policy can be applied to a complete L4 header — non-initial fragments lack the L4 ports needed for spoofing checks and are dropped if reassembly is not enabled or fails.

### 3.4. Deep Packet Inspection (DPI)
- **Metadata classification:** Fast, low assurance.
- **Full stream inspection:** Treat first-N-byte inspection as classification only, not a security boundary. Any DPI claim that blocks malicious L7 payloads must specify TCP reassembly, IP fragment handling, overlap policy, memory limits, and fail-closed behavior for ambiguous flows.

## 4. Packet Flow & Packet Ownership

1. **NIC RX/TX:** `FILL -> XSK_RX -> processing -> XSK_TX -> COMPLETION -> FILL`
2. **VM Ingress (Single-copy):** `XSK_RX UMEM frame -> copy into guest virtqueue buffer -> Vhost_TX completion`
3. **VM Egress (Single-copy):** `Vhost_RX guest buffer -> copy into UMEM frame -> XSK_TX -> COMPLETION`

Selected flows tagged `forward-to-proxy` are handed off to the (deferred) proxy delivery path described in §2; copy budget for that path will be defined when the mechanism is chosen.

## 5. Fail-Closed & Isolation Policy

- **Dataplane crash / XSK unbound:** XDP program defaults to `XDP_DROP`. The XDP program remains attached after process exit so traffic continues to drop until the dataplane is restarted.
- **Control-plane crash:** The last-published policy snapshot remains in effect; the dataplane continues processing under that snapshot. Policy reloads are blocked until the control plane is healthy. The dataplane does not fail open if the control plane disappears.
- **UMEM/Ring full:** Packet drop (head-drop or tail-drop). vhost-user backpressure to the guest is preferred over userspace drops; exact policy is a v1 open question (§7).
- **Physical Interface:** No host IP address; unreachable by the normal host stack.
- **AF_XDP fallback to copy/generic mode:** Production refuses to start unless explicitly permitted by config.
- **Execution:** Dataplane runs with minimal capabilities (drop `CAP_BPF`/`CAP_NET_ADMIN` after attach; retain `CAP_IPC_LOCK` for UMEM mlock), seccomp allowlist, strict cgroups. Control plane runs as a separate process with separate privileges.

## 6. Implementation Strategy (Revised Roadmap)

1. **Architecture Decision Record:** finalize network mode, interface mode, fail-closed policy, and offloads.
2. **Physical NIC AF_XDP proof:** single queue, native XDP zero-copy, bounded drops.
3. **VM I/O proof:** vhost-user path with single-copy measurement.
4. **Buffer lifecycle & backpressure:** prove TX-full behavior.
5. **Parser and anti-spoofing:** L2/L3/L4 parser, tenant identity enforcement.
6. **Stateless policy:** allow/drop with drop reasons.
7. **Conntrack & NAT:** tenant-aware state, timeouts, quotas.
8. **Proxy path:** proxy delivery mode and destination preservation.
9. **DPI classification:** metadata/prefix limits.
10. **Full DPI:** stream reassembly, evasion policy.
11. **Performance & Operational hardening.**

## 7. Open Questions (v1 ADR — Roadmap step 1)

The following must be resolved before roadmap step 5 (Parser/anti-spoofing). Each is referenced from the relevant section above.

- **Proxy / downstream-service forwarding mechanism** (§2): deferred to a future design pass — affects copy budget, original-destination metadata conveyance, and TCP/UDP scope. v1 only reserves the ACL action and conntrack hooks.
- **NAT model** (§2): SNAT, DNAT, or both; port allocation strategy; per-tenant vs shared port pool; ICMP/ICMPv6 error translation; hairpin NAT for VM-to-VM.
- **East-west (VM-to-VM) hairpin** (§2): hairpinned in dataplane vs round-tripped through the physical switch; same policy pipeline as north-south or fast intra-host path.
- **ARP / ND / DHCP plane** (§2): who answers ARP for the gateway IP, for other VM IPs (proxy-ARP), and for off-link IPs; RA emission policy; DHCP server vs relay vs neither; DHCPv4 and DHCPv6 separately.
- **Control-plane API** (§3): identity record schema (VM-ID, MACs, IPv4/IPv6, VLAN, queue/core, tenant, conntrack zone, rate limits), lifecycle (register/update/drain/deregister), authorization model (mTLS, capability tokens), versioning.
- **vhost-user feature mask** (§2): exact `VIRTIO_NET_F_*` and `VHOST_USER_PROTOCOL_F_*` bits negotiated in v1; `VIRTIO_F_IOMMU_PLATFORM` policy (GPA vs IOVA implications).
- **NIC / driver / kernel matrix:** tested combinations; behavior when AF_XDP falls back to copy mode.
- **Drop-reason enum:** stable enum for telemetry/logs; ABI versioning.
- **Slow-path threading:** ACL/conntrack-create/NAT-allocate inline on dataplane thread vs worker pool; DPI/proxy handoff queue and per-tenant fairness.
- **vhost-user backpressure:** stop pulling from guest virtqueue (recommended) vs userspace tail-drop.
- **Conntrack zone** (§3.2): defined per tenant (default), per VLAN, or per VM-ID.
- **Memory budgets:** bytes per flow × max flows × tenants; UMEM total; per-tenant share of conntrack/NAT/DPI buffers.
- **CPU isolation specifics:** `isolcpus`/`nohz_full`/`rcu_nocbs` baseline; IRQ pinning; `SCHED_FIFO` for dataplane threads.
- **Sandboxing posture:** capability list at runtime; seccomp profile; namespace strategy.
- **QUIC/UDP scope:** UDP/443 default-drop only, or QUIC handshake detection regardless of port; ICMP "fragmentation needed" handling to elicit TCP fallback.