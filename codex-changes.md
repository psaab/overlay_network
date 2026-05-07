# Adversarial Design Review and Proposed Changes

Reviewed documents:

- `DESIGN.md`
- `DETAILED_DESIGN.md`

Reviewer posture: assume hostile VMs, hostile tenants, malformed packets, driver limitations, queue exhaustion, proxy failure, CPU/NUMA pathologies, and attackers intentionally shaping traffic to hit the slowest or least specified path.

## Executive Assessment

The design has a reasonable high-level goal, but it currently overclaims end-to-end zero-copy, wire-speed DPI, VM isolation, and transparent proxy behavior. The largest issue is that it conflates three different data paths:

1. AF_XDP zero-copy on a supported physical NIC queue.
2. TAP/veth or virtio/vhost-user traffic from a VM.
3. Delivery to a normal local or external proxy.

Those paths do not automatically share a packet buffer model, offload model, or forwarding semantic. If the system is built as written, it is likely to become either a one-copy userspace bridge with partial security properties, or a complex custom vhost-user backend whose actual zero-copy constraints are much stricter than the docs imply.

The docs should be revised to pick an explicit architecture:

- **Recommended baseline:** AF_XDP zero-copy on physical NIC queues, vhost-user or TAP on the VM side, and an explicit single-copy boundary between guest virtqueues and AF_XDP UMEM. Optimize the copy with batching, hugepages, prefetch, and NUMA placement.
- **If end-to-end VM-to-NIC zero-copy is non-negotiable:** strongly consider DPDK vhost-user plus NIC PMDs, vDPA, or SR-IOV with tc/XDP/eBPF enforcement. AF_XDP can still be useful, but it is not the natural abstraction for passing arbitrary guest virtqueue buffers directly to a NIC.

## Blocker Design Corrections

### 1. Stop Claiming VM-to-NIC Zero-Copy Without a Precise Memory Contract

`DESIGN.md` says AF_XDP provides raw zero-copy I/O from both VM-facing and physical-facing interfaces. `DETAILED_DESIGN.md` says packets can move from `Vhost_RX` to `XSK_TX` by swapping descriptors. That is not generally correct.

AF_XDP TX descriptors reference offsets inside a registered AF_XDP UMEM. A vhost-user RX descriptor references guest memory exposed through QEMU/vhost-user memory tables. Those are different memory domains unless the design deliberately makes the guest packet buffers be AF_XDP UMEM-compatible DMA memory. That is a major architecture decision, not an implementation detail.

Proposed doc changes:

- Replace "zero-copy bridge between VM and physical network" with "zero-copy on supported NIC queues, with an explicit VM-side copy unless a stricter shared-memory design is selected."
- Delete or qualify "simply swap the descriptor from `Vhost_RX` to `XSK_TX`."
- Add a packet-buffer ownership diagram for:
  - NIC RX: `FILL -> XSK_RX -> processing -> XSK_TX -> COMPLETION -> FILL`
  - VM egress single-copy path: `Vhost_RX guest buffer -> copy into UMEM frame -> XSK_TX -> COMPLETION`
  - VM ingress single-copy path: `XSK_RX UMEM frame -> copy into guest virtqueue buffer -> Vhost_TX completion`
- State whether guest memory will ever be registered as AF_XDP UMEM. If yes, specify IOMMU, pinning, hugepage layout, chunk size, headroom, DMA safety, and how a hostile guest is prevented from corrupting host/NIC-visible buffers.

### 2. TAP/VETH, Vhost-User, and AF_XDP Need Separate Treatment

The docs mix TAP/VETH and vhost-user as if they are equivalent VM-facing interfaces. They are not.

TAP/veth traffic may be reachable through kernel devices and generic/native XDP paths, but this does not give the same zero-copy behavior as a physical NIC with AF_XDP zero-copy driver support. vhost-user is not a netdev at all; it is a userspace virtio backend protocol with memory-table negotiation, vrings, eventfds, feature bits, kicks, and calls.

Proposed doc changes:

- Split the VM-facing interface section into two mutually exclusive modes:
  - **TAP/veth mode:** simpler integration, usually copy-mode, easier namespace testing, lower peak performance.
  - **vhost-user backend mode:** higher performance potential, but requires implementing virtio-net backend semantics.
- For vhost-user, list required features and decisions:
  - split vs packed virtqueues
  - multiqueue support and queue-to-core mapping
  - `VIRTIO_NET_F_CSUM`, guest TSO/UFO/GSO, mergeable buffers, RSS, MAC, status, ctrl queue
  - eventfd kick/call handling
  - IOTLB and memory table updates
  - queue reset, VM disconnect, reconnect, and live migration policy
- For TAP/veth mode, remove any implication of guaranteed AF_XDP zero-copy.

### 3. Define the L2/L3 Product Mode Before Designing NAT and Proxying

The docs describe the system as a transparent bridge, an L2 switch, a NAT engine, and a transparent proxy. Those modes have different packet ownership and address semantics.

Proposed doc changes:

- Add a required "Network Mode" section with exactly one primary mode for v1:
  - **Transparent L2 bridge:** preserve VM IPs, forward Ethernet frames, handle ARP/ND/broadcast/multicast, no default L3 NAT.
  - **Routed/NAT gateway:** VMs are behind an overlay subnet, system owns gateway IP/MAC, NAT is first-class.
  - **Explicit proxy gateway:** terminate or relay selected flows, with clear proxy protocol semantics.
- Specify ARP/ND behavior:
  - gateway MAC ownership
  - ARP proxying or bridging
  - IPv6 neighbor discovery
  - router advertisements and DHCP
  - broadcast and multicast flooding limits
- Specify VM-to-VM hairpin behavior and whether the physical network ever sees east-west traffic.

### 4. Transparent Proxying Is Underspecified and Currently Unrealistic

Rewriting packets to a local proxy does not automatically work if the kernel stack is bypassed with AF_XDP. A normal Envoy/Squid process reading TCP sockets will not receive packets sitting in AF_XDP UMEM unless the design reinjects traffic into the kernel, connects the proxy through a veth/TAP namespace, or implements TCP termination in the userspace stack.

HTTPS DPI is also impossible without TLS termination, endpoint cooperation, or key material. For QUIC/HTTP/3, most payload and much metadata are encrypted.

Proposed doc changes:

- Add a "Proxy Delivery Path" section:
  - local proxy via veth/TAP into a proxy namespace
  - external proxy via normal L3 forwarding and NAT
  - in-process L4/TLS proxy
  - kernel TPROXY mode, if the design chooses to re-enter the kernel
- State how original destination metadata is preserved for the proxy.
- State TLS policy explicitly:
  - no decryption: only metadata/SNI/ALPN/JA3-like policy where available
  - TLS interception: requires enterprise CA, certificate generation, trust deployment, and legal/operational controls
  - endpoint-provided keys: limited deployment model
- Add an explicit QUIC/UDP/443 policy. Otherwise QUIC becomes the default bypass around HTTP/TLS inspection.

### 5. DPI Must Not Be Presented as Optional Reassembly

The docs say stream reassembly is optional and first-N-byte inspection is an alternative. That is only true for a weak classification product, not for a security product claiming DPI. Attackers can split signatures across TCP segments, use overlapping segments, IP fragmentation, IPv6 extension headers, chunking, compression, TLS, QUIC, or protocol upgrades.

Proposed doc changes:

- Reclassify DPI modes:
  - **Metadata classification:** fast, low assurance.
  - **Prefix inspection:** useful for protocol identification, not a security boundary.
  - **Full stream inspection:** requires TCP reassembly, flow buffering, timeouts, overlap policy, memory caps, and backpressure.
- Add mandatory evasion handling:
  - IPv4 fragments: reassemble before L4 policy or drop non-initial fragments by policy.
  - IPv6 extension headers and fragments: parse boundedly and drop ambiguous chains.
  - TCP overlap policy: document Linux-compatible or strict-drop behavior.
  - Out-of-order segments: buffer within a small bounded window or fail closed.
- Do not promote a flow to fast path merely because "valid HTTPS" was detected. Valid TLS can carry arbitrary exfiltration and tunnels.

### 6. Offloads, Segmentation, and Checksums Need a Real Contract

The design mentions incremental checksum updates but not the much larger issue: virtio and NIC offloads change what a "packet" means. Guests may transmit checksum-partial frames, TSO/GSO super-packets, UFO-like UDP segmentation, or receive-side mergeable buffers depending on negotiated features. NIC RX may involve checksum status metadata, and TX may require checksum offload support or software completion.

Proposed doc changes:

- Add an "Offload Contract" section:
  - which virtio-net features are negotiated or disabled
  - whether checksum-partial packets are accepted
  - whether TSO/GSO/GRO/LRO are disabled or implemented
  - maximum MTU and jumbo-frame policy
  - AF_XDP multi-buffer support policy
  - VLAN tag handling, including hardware-stripped VLAN metadata
- For v1, strongly consider disabling guest TSO/GSO and checksum partial, then enabling them only after correctness tests exist.
- Specify that every header rewrite updates:
  - Ethernet FCS is not present in host memory
  - IPv4 header checksum
  - TCP/UDP checksum including pseudo-header
  - checksum-zero behavior for IPv4 UDP
  - IPv6 UDP checksum requirements

### 7. "Completely Isolated" Needs Fail-Closed Bypass Prevention

The executive summary says the production network remains completely isolated and protected from untrusted VM traffic. That is an absolute claim, and the current design does not specify the failure behavior needed to justify it.

Proposed doc changes:

- Add a fail-closed section for:
  - XSK not bound
  - AF_XDP ring full
  - UMEM depletion
  - userspace process crash
  - XDP program load or attach failure
  - DPI engine failure
  - proxy unavailable
  - config reload failure
  - unsupported NIC driver or fallback to copy/generic mode
- The XDP program should default to `XDP_DROP` when redirect is impossible or no socket is configured, unless the product explicitly allows fail-open.
- Ensure the physical NIC has no host IP address and is not reachable through the normal host stack unless deliberately configured.
- Run the dataplane with minimum capabilities after setup, seccomp, locked-down control socket permissions, cgroup limits, and explicit crash recovery.

## High-Priority Design Changes

### 8. Replace MAC Learning With Provisioned Anti-Spoofing

The detailed design proposes MAC learning by observing VM source MACs. In a hostile VM model, MAC learning is an attack surface. A VM can claim another VM's MAC, the gateway MAC, multicast addresses, or physical-network addresses.

Proposed doc changes:

- Use provisioned VM identity: VM-ID maps to allowed MACs, IPs, VLANs, and optional allowed DHCP leases.
- Enforce egress anti-spoofing before conntrack:
  - source MAC
  - source IPv4/IPv6
  - VLAN tags
  - ARP sender protocol/hardware address
  - IPv6 neighbor advertisement source
  - DHCP client identity if used
- Treat learning as telemetry only, not authority.
- Add ARP guard, DHCP guard, IPv6 RA guard, and ND guard.

### 9. Conntrack Needs Tenant, Direction, NAT, and Protocol State

A simple 5-tuple hash table is insufficient for multi-tenant NAT/proxy enforcement.

Proposed doc changes:

- Define the flow key as at least:
  - tenant or VM-ID
  - ingress interface and queue
  - direction
  - L3 protocol
  - normalized 5-tuple
  - conntrack zone
- Store NAT/proxy tuple bindings separately:
  - original tuple
  - translated tuple
  - reverse tuple
  - proxy association
  - timeout class
- Add protocol-specific state:
  - TCP SYN/SYN-ACK/ACK establishment
  - FIN/RST close
  - idle timeouts
  - sequence-window sanity checks where feasible
  - UDP pseudo-state with short timeouts
  - ICMP and ICMPv6 error relation to existing flows
- Add memory-exhaustion policy: per-tenant quotas, global caps, eviction priority, SYN flood handling, and metrics.

### 10. Queue Ownership and Scaling Must Be Explicit

AF_XDP is queue-oriented. High performance depends on a clean mapping of NIC RX/TX queues, XSKs, UMEM rings, vhost virtqueues, and CPU cores. The docs only say "lock-less" and "SPSC", which is not enough.

Proposed doc changes:

- Add a queue topology:
  - one XSK per NIC queue
  - one dataplane thread per queue pair or per queue group
  - one UMEM pool per NUMA node or per queue group
  - one vhost virtqueue pair per VM queue
  - explicit VM queue to dataplane core assignment
- Avoid cross-core packet handoff on the fast path. If needed, use RSS/RPS-like steering before the packet enters userspace.
- Define TX backpressure:
  - what happens when physical TX ring is full
  - how guest virtqueues are stopped or delayed
  - when packets are dropped
  - whether drops are head-drop, tail-drop, or priority based
- Add `XDP_USE_NEED_WAKEUP`, busy-poll, interrupt moderation, and syscall wakeup behavior to the AF_XDP section.

### 11. NUMA, Cache, and CPU Claims Need Quantified Budgets

"Wire speed" is not a single target. At minimum, the docs should name packet rates, packet sizes, and CPU budget.

Proposed doc changes:

- Add target envelopes:
  - 10 GbE: 14.88 Mpps at 64-byte Ethernet frames
  - 25 GbE: 37.2 Mpps at 64-byte Ethernet frames
  - 100 GbE: 148.8 Mpps at 64-byte Ethernet frames
  - target p50/p99 latency under load
  - number of cores and NIC queues
- Add CPU budget per packet for each path:
  - L2 fast path
  - L3/L4 ACL path
  - conntrack miss
  - NAT rewrite
  - DPI prefix inspection
  - full stream inspection
- Add cache-layout requirements:
  - cacheline-align hot counters and rings
  - avoid false sharing in flow table shards
  - per-core allocator/cache for packet metadata
  - hugepages for UMEM and vhost memory
  - NIC/CPU NUMA affinity and IRQ affinity
- Note that software prefetch must be validated; bad prefetch distance can hurt more than help.

### 12. Memory Isolation Is Not Achieved by UMEM Slicing Alone

Partitioning a single process-owned UMEM into VM-specific ranges is a convenience, not a hard isolation boundary. A bug in unsafe Rust, BPF map handling, FFI, pointer arithmetic, or descriptor validation can corrupt any range. If guest memory is mapped into the process, the blast radius grows.

Proposed doc changes:

- Call UMEM slicing "logical isolation", not memory isolation.
- Add hardening options:
  - separate process per tenant or per trust zone
  - separate UMEM per VM or per trust zone
  - IOMMU and device isolation where applicable
  - strict descriptor bounds checks
  - never trust guest-provided lengths, offsets, checksum flags, or header positions
  - fuzz all parsers and virtqueue descriptor walkers
- Define whether shared physical NIC queues are allowed across tenants and how per-tenant accounting is preserved.

### 13. BPF/XDP Program Behavior Needs Specification

The design says "redirect `XDP_PASS` to `XDP_REDIRECT`," which is not how XDP actions are modeled. The program returns one action per packet. It also omits what happens when the XSK map has no socket for a queue.

Proposed doc changes:

- Add XDP program pseudocode:
  - read `rx_queue_index`
  - look up XSK in `xsks_map`
  - return `bpf_redirect_map(...)`
  - return `XDP_DROP` if no socket is active in fail-closed mode
  - optionally allow `XDP_PASS` only in explicitly configured diagnostics mode
- State supported attach modes:
  - native driver XDP required for AF_XDP zero-copy
  - generic XDP allowed only for development or fallback
  - hardware offload not assumed unless tested
- Include libxdp/multi-program attach policy if coexisting with other XDP programs.

### 14. Observability Must Include Queue and Failure State

Prometheus counters are useful, but the current telemetry list is too generic for this class of system.

Proposed doc changes:

- Add metrics for:
  - RX/TX ring occupancy per queue
  - fill/completion ring starvation
  - `XDP_REDIRECT` failures
  - UMEM allocation failures
  - per-stage drop reasons
  - flow table occupancy and eviction
  - NAT table occupancy
  - per-tenant packet/byte/drop counters
  - DPI bytes inspected, flows bypassed, flows failed closed
  - proxy connect failures and latency
  - p50/p95/p99/p999 latency by path
  - CPU cycles per packet, LLC misses, branch misses during benchmarks
- Add a structured drop-reason enum in the dataplane design. Debuggability will otherwise be poor under attack.

### 15. Control Plane Reload Needs RCU/Epoch Semantics

An atomic pointer swap for global config is plausible, but reclamation and in-flight packet consistency must be specified.

Proposed doc changes:

- Use versioned immutable policy snapshots.
- Publish new snapshots with atomic pointer swap.
- Reclaim old snapshots using epoch/RCU after all dataplane threads have advanced.
- Validate config before publication.
- Make reload transactional: reject invalid policies without changing dataplane behavior.
- Include rollback and audit log behavior.

## Security and Correctness Additions

### 16. Parser Requirements

Add a parser contract before firewall/DPI:

- Bounds-check every header access.
- Parse VLAN/QinQ with bounded depth.
- Parse IPv4 options or drop packets with options in strict mode.
- Parse IPv6 extension headers with bounded chain length.
- Handle non-first fragments explicitly.
- Validate TCP data offset and flags.
- Validate UDP length.
- Validate ICMP/ICMPv6 type/code and quoted packet length.
- Normalize or drop malformed packets before conntrack.

### 17. Rate Limiting and Scheduling

Per-VM token buckets are a good start, but they need placement and semantics.

Proposed doc changes:

- Apply rate limits before shared TX queues to protect other tenants.
- Use byte and packet rate limits; pps limits matter for CPU exhaustion.
- Add control traffic priority for ARP/ND/DHCP if applicable, with caps to prevent abuse.
- Define fairness when multiple VMs share one physical TX queue.
- Include per-tenant burst limits and maximum concurrent slow-path flows.

### 18. Testing Plan Is Too Weak

Network namespaces and veth pairs are useful but cannot validate AF_XDP zero-copy, NIC offloads, PCIe/NUMA behavior, or vhost-user correctness.

Proposed doc changes:

- Add test layers:
  - parser fuzzing with libFuzzer/AFL/cargo-fuzz
  - pcap replay for malformed and evasive traffic
  - QEMU vhost-user integration tests
  - netns/veth functional tests
  - physical NIC AF_XDP zero-copy tests per driver
  - TRex/MoonGen/pktgen throughput and latency tests
  - chaos tests for process crash, proxy outage, ring exhaustion, and config reload
- Add adversarial test cases:
  - TCP segmentation evasion
  - overlapping TCP segments
  - IPv4/IPv6 fragments
  - IPv6 extension header chains
  - QUIC over UDP/443
  - ARP/ND spoofing
  - DHCP spoofing
  - VLAN hopping/QinQ
  - flow table exhaustion
  - TX ring full under mixed tenants

### 19. Rust Safety Claims Need Unsafe-Boundary Design

Rust helps, but this system necessarily uses unsafe code: mmap rings, UMEM pointer arithmetic, BPF FFI, vhost-user memory mapping, shared memory, and possibly Hyperscan FFI.

Proposed doc changes:

- Add an unsafe boundary inventory.
- Wrap packet buffers in types that carry validated length and headroom.
- Make descriptor validation explicit before converting offsets to slices.
- Forbid raw pointer arithmetic outside a small module.
- Add Miri where possible for pure logic, plus fuzzing for unsafe parser boundaries.
- Audit external crates used for AF_XDP, vhost-user, and Hyperscan bindings.

### 20. Hyperscan and CPU Feature Policy Need Specifics

Hyperscan-class DPI is not just a library call. It has per-thread scratch requirements, database compilation choices, CPU feature assumptions, and possible throughput cliffs when patterns force expensive automata or when AVX-512 changes core frequency behavior.

Proposed doc changes:

- Specify whether pattern databases are compiled offline or at runtime.
- Allocate one Hyperscan scratch region per dataplane thread; do not share scratch on the fast path.
- Define CPU feature targets:
  - SSE4.2/AVX2 baseline
  - AVX-512 allowed or disabled, depending on measured frequency impact
  - fallback behavior on older CPUs
- Add a rule admission policy: reject or quarantine regexes that explode database size, compile time, or scan cost.
- Track DPI cost metrics by ruleset version.
- Treat DPI workers as a separate CPU budget from L2/L3 fast-path workers unless prefix inspection is proven cheap enough inline.

### 21. Kernel, Driver, and NIC Compatibility Must Be a Gate

The docs say Linux 5.10+ and 6.x recommended, but AF_XDP zero-copy behavior is driver-specific and feature-specific. A kernel version alone is not a compatibility guarantee.

Proposed doc changes:

- Add a tested matrix:
  - kernel version
  - NIC model and firmware
  - driver
  - XDP native support
  - AF_XDP zero-copy support
  - AF_XDP multi-buffer support
  - checksum/offload behavior
  - maximum queues tested
- At startup, detect and report:
  - whether the XSK is actually in zero-copy or copy mode
  - attach mode used by XDP
  - active NIC offloads
  - ring sizes and queue count
  - NUMA node for the PCI device
- Refuse production mode if the runtime path falls back to generic XDP or AF_XDP copy mode unless policy explicitly permits it.

### 22. Timeouts and Timers Need a Per-Core Design

Conntrack, NAT, DPI buffering, token buckets, and retransmission-related logic all need timers. A global timer heap will become a scalability and contention problem.

Proposed doc changes:

- Use per-core timer wheels or sharded timeout structures.
- Keep flow ownership stable so the same core updates the flow and its timer state.
- Define clock source assumptions:
  - invariant TSC availability
  - behavior across CPU migration
  - fallback to `clock_gettime` or vDSO
- Bound cleanup work per poll-loop iteration to avoid latency spikes.
- Add explicit timeout classes for TCP established, TCP closing, UDP, ICMP, fragments, half-open flows, DPI-pending flows, and proxy-pending flows.

## Roadmap Changes

The current roadmap moves to firewalling and DPI before proving the hard I/O and correctness boundaries. Reorder it.

Proposed roadmap:

1. **Architecture decision record:** choose VM interface mode, network mode, proxy mode, fail-open/fail-closed policy, and offload contract.
2. **Physical NIC AF_XDP proof:** one queue, native XDP, zero-copy verified by driver stats, bounded drop behavior.
3. **VM I/O proof:** vhost-user or TAP/veth path with explicit copy/zero-copy measurement and feature negotiation.
4. **Buffer lifecycle and backpressure:** prove no descriptor reuse before completion; define TX-full behavior.
5. **Parser and anti-spoofing:** L2/L3/L4 parser, ARP/ND/DHCP guard, tenant identity enforcement.
6. **Stateless policy:** allow/drop with structured drop reasons and metrics.
7. **Conntrack and NAT:** tenant-aware state, timeouts, quotas, reverse mappings, ICMP relation.
8. **Proxy path:** choose delivery mode, preserve original destination, handle proxy failure.
9. **DPI classification:** start with metadata/prefix classification and clear assurance limits.
10. **Full DPI, if required:** reassembly, evasion policy, memory caps, TLS/QUIC policy.
11. **Performance hardening:** multiqueue scaling, NUMA tuning, CPU isolation, benchmark gates.
12. **Operational hardening:** seccomp, capabilities, watchdogs, transactional config, crash behavior.

## Suggested Text Replacements

Replace:

> secure, near zero-copy bridge between a Virtual Machine and the physical network

With:

> secure userspace mediation layer between VM virtio traffic and physical NIC queues. The physical NIC path uses AF_XDP zero-copy where the driver supports it; the VM boundary is explicitly modeled as single-copy unless a separately specified shared-memory design is selected.

Replace:

> For raw, high-throughput, zero-copy packet I/O from both the VM-facing interface and the physical-facing interface.

With:

> For raw, high-throughput packet I/O on netdev queues. Zero-copy is available only for supported native-XDP NIC drivers and only for buffers registered in AF_XDP UMEM. VM-facing TAP/veth and vhost-user paths have separate copy and feature-negotiation constraints.

Replace:

> we simply swap the descriptor from the `Vhost_RX` ring to the `XSK_TX` ring

With:

> vhost-user descriptors and AF_XDP descriptors reference different memory domains. The baseline implementation copies packet bytes from guest buffers into AF_XDP UMEM on egress and from AF_XDP UMEM into guest buffers on ingress. Descriptor passing is allowed only if the selected architecture makes guest packet buffers valid AF_XDP UMEM frames and satisfies DMA isolation requirements.

Replace:

> Implement stream reassembly (optional but necessary for complex L7 inspection across packet boundaries). If full reassembly is too slow, implement first-N-bytes packet inspection.

With:

> Treat first-N-byte inspection as classification only, not a security boundary. Any DPI claim that blocks malicious L7 payloads must specify TCP reassembly, IP fragment handling, overlap policy, memory limits, and fail-closed behavior for ambiguous flows.

Replace:

> Automatically populates the Steering Table by observing the source MAC of packets exiting each `vhost-user` interface.

With:

> Populate the Steering Table from control-plane VM identity. Observed source MACs may be logged, but untrusted VM traffic must not authoritatively update forwarding ownership. Enforce provisioned MAC/IP/VLAN bindings before conntrack.

## Minimum Acceptance Criteria for the Revised Design

Before implementation starts, the docs should be able to answer these questions unambiguously:

- Is v1 TAP/veth or vhost-user?
- Is v1 L2 bridge, routed/NAT gateway, or proxy gateway?
- Which exact traffic path is zero-copy, and which path copies?
- What happens when AF_XDP redirect fails?
- What happens when the userspace process dies?
- Which virtio-net offloads are negotiated?
- What is the MTU and multi-buffer policy?
- How are guest MAC/IP/VLAN identities provisioned and enforced?
- How does a local proxy receive packets if the kernel stack is bypassed?
- Is HTTPS decrypted, classified by metadata only, or not inspected?
- What is the QUIC policy?
- Are IPv4/IPv6 fragments reassembled or dropped?
- What are the per-tenant flow, packet, byte, and slow-path limits?
- What packet rate, latency, CPU count, NIC, and driver define "line rate"?
- How is old policy memory reclaimed after atomic config swaps?
- Which unsafe Rust modules exist, and how are they fuzzed?
