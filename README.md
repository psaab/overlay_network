# XDP-Prox: High-Performance Userspace Networking Stack

XDP-Prox is a high-performance, userspace networking stack designed to act as a secure mediation layer between Virtual Machine (VM) virtio traffic and physical NIC queues.

Leveraging **AF_XDP** for zero-copy I/O on supported physical NICs and a strict single-copy boundary for VM virtio interfaces (via `vhost-user`), XDP-Prox targets a 10 GbE envelope (14.88 Mpps at 64-byte frames; see [DETAILED_DESIGN.md §5](DETAILED_DESIGN.md)) for stateless firewalling, with DPI classification (metadata only in v1) and selective proxy forwarding, all while maintaining a rigorous fail-closed security posture.

**v1 is IPv6-only.** The physical network is IPv6-only and VMs receive globally-routable IPv6 addresses. The dataplane performs pure L3 IPv6 forwarding without rewriting source or destination addresses — there is no NAT/DNAT in the forwarding plane. IPv4 connectivity is planned via encapsulation (4in6 / MAP-T / DS-Lite class) in a future design revision and is out of scope for v1.

## Key Features

- **AF_XDP Zero-Copy on Supported NICs:** Native-driver XDP with AF_XDP zero-copy on tested NICs (Intel E810, Mellanox CX-6); copy-mode and generic XDP exist as CI/dev fallbacks only and are refused in production by default.
- **Strict Isolation Boundary:** v1 uses a single-copy boundary at the VM interface; guest memory is never DMA-mapped to the NIC. Logical UMEM slicing and strict descriptor bounds checking apply to host-owned frames.
- **Anti-Spoofing & Steering:** L2/L3 identities are control-plane provisioned. v1 enforces MAC, IPv6, and VLAN bindings before conntrack. ND/RA guard is enforced on the VM-facing side: VM-sourced router advertisements are dropped, and VM-sourced NS/NA must match provisioned identity. (DHCPv6 guard depends on the chosen address-assignment plane — see DESIGN.md §7.)
- **Fail-Closed Security:** Unbound XSK and dataplane process exit return `XDP_DROP` at the BPF program level; missing config, parse failures, ring exhaustion, and ACL deny result in userspace drops with structured drop reasons.
- **IPv6 Routing Gateway with Selective Proxy:** Pure IPv6 L3 forwarding; no NAT/DNAT on the v1 hot path. Selected flows (per ACL) are tagged `forward-to-proxy`; the forwarding mechanism is deferred and tagged flows are dropped in v1 with reason `proxy_deferred` until the mechanism is chosen. IPv4 traffic from VMs is also dropped in v1, pending the future encapsulation design.
- **Multi-Tenant Aware:** Flow classification includes tenant and VM IDs to enforce fairness and quotas.

## Architecture

XDP-Prox employs a split-plane architecture:
- **Data Plane (Rust):** High-speed poll loop running on isolated CPUs. Handles AF_XDP ring management, vhost-user queues, fast-path conntrack, and parsing.
- **Control Plane:** Manages configuration via Epoch/RCU-based snapshots to allow hitless policy reloads.

For full architectural details, read:
- [DESIGN.md](DESIGN.md) - Executive overview, packet flow, and v1 open questions.
- [DETAILED_DESIGN.md](DETAILED_DESIGN.md) - Engineering specifications, queue topologies, and performance budgets.
- [codex-changes.md](codex-changes.md) - Historical adversarial review (non-normative). Findings have been incorporated into the design docs above; preserved here as a record.

## Roadmap (summary)

The canonical roadmap is in [DESIGN.md §6](DESIGN.md). Headline phases:

1. Architecture Decision Record (ADR) — resolves v1 open questions
2. Physical NIC AF_XDP proof (zero-copy verified, bounded drops)
3. VM I/O proof (vhost-user single-copy)
4. Buffer lifecycle & backpressure
5. Parsers & Anti-Spoofing
6. Conntrack (no NAT in v1)
7. Proxy Delivery Path
8. DPI Metadata Classification
9. Full DPI (if required)
10. Performance hardening
11. Operational hardening

## Development & Security Posture
XDP-Prox is implemented in Rust, utilizing unsafe boundaries explicitly where interfacing with memory-mapped rings, BPF maps, or FFI boundaries. It mandates disabled virtio-net offloads (TSO/GSO) to enforce strict packet constraints before processing.
