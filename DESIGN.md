# Userspace Networking Stack Architecture (AF_XDP)

## 1. Executive Summary
This document outlines the design and implementation strategy for a high-performance, userspace networking stack. The goal is to provide a secure userspace mediation layer between VM virtio traffic and physical NIC queues. The physical NIC path uses AF_XDP zero-copy where the driver supports it; the VM boundary is explicitly modeled as single-copy unless a separately specified shared-memory design is selected. The stack enforces deep packet inspection (DPI), wire-speed firewalling, and transparent proxy forwarding, ensuring the physical production network remains protected from untrusted VM traffic.

## 2. Architecture Overview

The system sits as a transparent, high-speed mediator between the VM's network interface and the physical uplink.

### Network Mode
The primary mode for v1 is an **Explicit proxy gateway**. The system will route/NAT flows, terminating or relaying selected flows with clear proxy protocol semantics.
- Gateway MAC is owned by the system.
- ARP proxying and IPv6 ND are explicitly managed.

### Core Technologies
- **AF_XDP Sockets (XSK):** For raw, high-throughput packet I/O on netdev queues. Zero-copy is available only for supported native-XDP NIC drivers and only for buffers registered in AF_XDP UMEM. VM-facing TAP/veth and vhost-user paths have separate copy and feature-negotiation constraints.
- **eBPF/XDP Programs:** Minimal eBPF programs attached to the physical interface whose sole purpose is to redirect packets into the AF_XDP sockets (`XDP_REDIRECT`), or `XDP_DROP` if the socket/queue is unconfigured.
- **Rust (Recommended):** For memory safety, aggressive concurrency (lock-free structures), and high performance.

### VM-Facing Interface
We explicitly split support into two mutually exclusive modes (v1 will target vhost-user):
- **vhost-user backend mode:** High performance, requires implementing virtio-net backend semantics (split virtqueues, multiqueue, eventfd kick/calls).
- **TAP/veth mode (Fallback/Testing):** Simpler integration, copy-mode, lower peak performance.

### Proxy Delivery Path
To deliver packets to a local proxy:
- External/local proxy via normal L3 forwarding and NAT, or veth/TAP into a proxy namespace.
- TLS Interception is not assumed. Without keys, DPI is limited to metadata (SNI/ALPN).
- Default DROP policy for QUIC (UDP/443) to prevent TLS inspection bypass.

## 3. Key Components

### 3.1. I/O Engine
Manages Fill, Completion, RX, and TX rings. The physical side uses AF_XDP, while the VM side uses `vhost-user`. 

### 3.2. Fast-Path Connection Tracker (Conntrack)
A flow table tracking tenant/VM-ID, ingress interface, direction, L3 protocol, and normalized 5-tuple. It maintains full TCP state (SYN/ACK/FIN/RST) to prevent state exhaustion.

### 3.3. Firewall Rules Engine
Evaluates packets against ACLs. Enforces anti-spoofing before conntrack (validating source MAC, IP, VLAN against provisioned VM identity).

### 3.4. Deep Packet Inspection (DPI)
- **Metadata classification:** Fast, low assurance.
- **Full stream inspection:** Treat first-N-byte inspection as classification only, not a security boundary. Any DPI claim that blocks malicious L7 payloads must specify TCP reassembly, IP fragment handling, overlap policy, memory limits, and fail-closed behavior for ambiguous flows.

## 4. Packet Flow & Packet Ownership

1. **NIC RX:** `FILL -> XSK_RX -> processing -> XSK_TX -> COMPLETION -> FILL`
2. **VM Ingress (Single-copy):** `XSK_RX UMEM frame -> copy into guest virtqueue buffer -> Vhost_TX completion`
3. **VM Egress (Single-copy):** `Vhost_RX guest buffer -> copy into UMEM frame -> XSK_TX -> COMPLETION`

If a packet requires proxying, it is pushed to the proxy namespace, not immediately forwarded.

## 5. Fail-Closed & Isolation Policy

- **Crash/XSK unbound:** XDP program defaults to `XDP_DROP`.
- **UMEM/Ring full:** Packet drop (head-drop or tail-drop).
- **Physical Interface:** No host IP address; unreachable by host stack.
- **Execution:** Dataplane runs with minimal capabilities, seccomp, strict cgroups.

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