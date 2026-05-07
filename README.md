# XDP-Prox: High-Performance Userspace Networking Stack

XDP-Prox is a high-performance, userspace networking stack designed to act as a secure mediation layer between Virtual Machine (VM) virtio traffic and physical NIC queues. 

Leveraging **AF_XDP** for zero-copy I/O on supported physical NICs and a strict single-copy boundary for VM virtio interfaces (via `vhost-user`), XDP-Prox provides line-rate firewalling, Deep Packet Inspection (DPI) classification, and transparent proxy forwarding, all while maintaining a rigorous fail-closed security posture.

## Key Features

- **AF_XDP Native Performance:** Utilizes `XDP_REDIRECT` for bypass of the host kernel networking stack.
- **Strict Isolation Boundary:** Explicit memory contracts separate VM and host memory domains, preventing DMA corruption from hostile guests.
- **Anti-Spoofing & Steering:** L2/L3 identities are control-plane provisioned, blocking ARP/ND/DHCP spoofing and MAC impersonation.
- **Fail-Closed Security:** Hardware constraints, unparsed headers, and system failures default to `XDP_DROP`.
- **Transparent Proxy Gateway:** NAT and header rewriting capabilities to redirect traffic to explicit or transparent inspection proxies.
- **Multi-Tenant Aware:** Flow classification includes tenant and VM IDs to enforce fairness and quotas.

## Architecture

XDP-Prox employs a split-plane architecture:
- **Data Plane (Rust):** High-speed poll loop running on isolated CPUs. Handles AF_XDP ring management, vhost-user queues, fast-path conntrack, and parsing.
- **Control Plane:** Manages configuration via Epoch/RCU-based snapshots to allow hitless policy reloads.

For full architectural details, read:
- [DESIGN.md](DESIGN.md) - Executive overview and packet flow.
- [DETAILED_DESIGN.md](DETAILED_DESIGN.md) - Engineering specifications, queue topologies, and performance budgets.
- [codex-changes.md](codex-changes.md) - Adversarial review and threat-model considerations that shaped the current architecture.

## Roadmap

1. Architecture Decision Record (ADR)
2. Physical NIC AF_XDP proof (Bounded Drops)
3. VM I/O proof (vhost-user single-copy)
4. Parsers & Anti-Spoofing
5. Conntrack & NAT
6. Proxy Delivery Path
7. DPI Metadata Classification

## Development & Security Posture
XDP-Prox is implemented in Rust, utilizing unsafe boundaries explicitly where interfacing with memory-mapped rings, BPF maps, or FFI boundaries. It mandates disabled virtio-net offloads (TSO/GSO) to enforce strict packet constraints before processing.
