# Design Review and Proposed Changes

Reviewed:
- `README.md`
- `DESIGN.md`
- `DETAILED_DESIGN.md`
- `codex-changes.md`

The codex pass already raised most of the structural problems and the two design docs have absorbed the majority of those corrections. This review focuses on what is still missing, what is now internally inconsistent after the rewrite, and where the docs are precise enough to read but not yet precise enough to implement.

The repo currently has a single commit (`ebcf4df`) and the design docs are unchanged from my prior pass. The new file is `README.md`, and it has drifted from both `DESIGN.md` and `DETAILED_DESIGN.md` — see section A0 below. Treat README/DESIGN/DETAILED_DESIGN as one corpus that has to agree before anyone implements against it.

## Overall Assessment

The design is *directionally* solid. After the codex revisions:

- The zero-copy story is honest (NIC zero-copy, single-copy at the VM boundary).
- Network mode is committed (explicit proxy gateway, vhost-user as v1).
- Anti-spoofing-before-conntrack and provisioned identity are correct.
- Fail-closed posture is stated end-to-end.
- The roadmap front-loads I/O and correctness over DPI, which is right.

However, the docs are still operating at the level of *intent* in several places where they need to be at the level of *contract*. A handful of contradictions also slipped in during the rewrite. None are fatal, but they should be resolved before anyone starts coding step 2 of the roadmap.

## A. Contradictions Introduced by the Rewrite (fix first)

These are inconsistencies between or within the current docs. They block the ADR-level decisions in roadmap step 1.

### A0. README.md disagrees with DESIGN.md on multiple load-bearing claims

The README is now the first thing a contributor reads, and it contradicts the design docs in five places:

1. **Proxy mode.** README §Key Features calls v1 a "Transparent Proxy Gateway." `DESIGN.md §2` calls it an "Explicit proxy gateway." Transparent and explicit proxies are different products with different client behavior, different conntrack semantics, and different L7 visibility. Pick one. (See section A2 — this is the same argument resolving across both files.)

2. **Memory isolation overclaim.** README §Key Features says "Explicit memory contracts separate VM and host memory domains, preventing DMA corruption from hostile guests." `DETAILED_DESIGN.md §2.1` is more honest: the v1 baseline is single-copy, which trivially prevents DMA corruption *because there is no guest DMA path*. The interesting case — guest buffers registered as AF_XDP UMEM — is explicitly out of v1. The README phrasing implies a mitigation against a threat the v1 design does not actually expose itself to, and will mislead anyone evaluating the security model. Reword to: "v1 uses a strict single-copy boundary; guest memory is never DMA-mapped to the NIC."

3. **DHCP-spoofing claim is unsupported.** README says anti-spoofing blocks "ARP/ND/DHCP spoofing and MAC impersonation." Neither `DESIGN.md` nor `DETAILED_DESIGN.md` actually specifies a DHCP guard. This is item B4 in the original review — the design needs to add it before the README can claim it.

4. **AF_XDP vs native XDP conflation.** README says "AF_XDP Native Performance: Utilizes XDP_REDIRECT for bypass of the host kernel networking stack." AF_XDP via `XDP_REDIRECT` works in *copy mode* too; only native-driver XDP with a zero-copy-capable driver gives the "no kernel stack" performance the bullet implies. Reword: "Native-driver XDP with AF_XDP zero-copy on supported NICs; copy-mode and generic XDP are dev/CI fallbacks only."

5. **Roadmap drift.** README lists 7 roadmap steps; `DESIGN.md §6` lists 11. The README ends at "DPI Metadata Classification" and silently drops "Buffer lifecycle & backpressure," "Full DPI," "Performance hardening," and "Operational hardening." Either the README is the marketing summary and should say so, or it is the canonical roadmap and `DESIGN.md` is wrong. Pick one source of truth.

These five items are easy fixes (1–2 hour edit pass) but they are blocking — anyone reading top-down will get a different mental model than anyone reading the design docs directly.

### A1. UMEM pool: one global, or one per NUMA node?

`DETAILED_DESIGN.md §2.1` says:
> Global Pool: Shared for physical NIC I/O.

`DETAILED_DESIGN.md §3.1` says:
> one UMEM pool per NUMA node.

These cannot both be true unless the system is single-NUMA. Pick one and state it:

- For a single-socket box, "one UMEM pool, per-queue chunks" is fine.
- For multi-socket, you need one UMEM per NUMA node, NIC IRQ affinitized to the local node, and a documented rule for which UMEM a flow's frames live in.

### A2. "Explicit proxy gateway" but also routing and NAT

`DESIGN.md §2` declares the v1 mode "Explicit proxy gateway" and immediately says "The system will route/NAT flows, terminating or relaying selected flows." That is three modes glued together. Either:

- Call it a "routed gateway with selective proxy," and define routing/NAT as primary with proxying as one of several next-hops, or
- Call it a "proxy gateway," and remove the "route/NAT" phrasing — every flow then goes through proxy semantics.

Right now the document does not commit, which leaves NAT, hairpin, and proxy delivery undefined.

### A3. Single-copy claim in §4 conflicts with proxy delivery in §2

`DESIGN.md §4` defines the VM ingress/egress paths as single-copy between UMEM and guest virtqueues. But `DESIGN.md §2 Proxy Delivery Path` says proxied packets go via "veth/TAP into a proxy namespace" — that path involves at least one additional copy into the kernel, then back out, and likely a second pass through the userspace stack on the return leg. The packet-flow section should add a third path:

```
VM-to-Proxy (proxied):
  Vhost_RX guest buffer
    -> copy into UMEM frame
    -> policy/conntrack
    -> copy/inject into proxy delivery iface (veth/TAP)
    -> kernel stack into proxy
    -> proxy egress (separate connection)
    -> back through dataplane on the new flow
```

Without this, readers think proxied flows are also single-copy, which they are not.

### A4. "First-N-byte inspection is classification, not security" but the firewall section still treats DPI as a security boundary

`DESIGN.md §3.4` correctly downgrades prefix inspection. But `DESIGN.md §3.3` and the roadmap still imply that DPI gates malicious L7. Add a one-line sentence in §3.3 that the firewall enforces L3/L4 policy and identity; L7 enforcement requires the full-stream DPI mode and is explicitly out of scope for prefix-only deployments.

## B. Required Specifications Still Missing

These are gaps the codex review flagged that the rewrite acknowledged but did not actually fill in.

### B1. Proxy delivery path: pick one for v1

`DESIGN.md §2` lists "External/local proxy via normal L3 forwarding and NAT, or veth/TAP into a proxy namespace." Two modes is two modes. Pick one for v1, and specify:

- How original destination (pre-NAT) is preserved for the proxy. Options: TPROXY + `SO_ORIGINAL_DST`-equivalent lookup against the conntrack table; PROXY-protocol v2 header prepended on the connection to the proxy; out-of-band flow metadata channel. **Recommend PROXY-protocol v2** for a userspace-bypass design — it does not require kernel TPROXY, works across namespaces, and is supported by Envoy/HAProxy/NGINX out of the box.
- How the return leg from the proxy reaches the original client. State whether the proxy terminates the client connection (so return = new connection from proxy back through the dataplane) or relays L4 (in which case the dataplane must NAT the proxy-sourced response onto the original 5-tuple).
- Where the proxy's egress traffic to the upstream comes back into the dataplane and how it is policy-evaluated a second time without infinite-looping.

### B2. NAT model is undefined

`DESIGN.md §2` says "route/NAT" but never defines:

- SNAT, DNAT, or both.
- Port allocation strategy (linear, randomized, hash-based) and exhaustion behavior.
- Per-tenant port pool versus shared pool, and how exhaustion in a shared pool is contained per-tenant.
- Whether ICMP/ICMPv6 errors are translated and tied back to the originating flow.
- Hairpin NAT behavior for VM-to-VM traffic that egresses through the proxy.

Add an "§3.x NAT Engine" subsection in `DETAILED_DESIGN.md`.

### B3. VM-to-VM (east-west) hairpin

The codex review called this out and the rewrite did not address it. Two questions for v1:

- Does east-west traffic ever leave the userspace dataplane? (Recommend: no — hairpin in the dataplane to keep policy/conntrack authoritative and avoid round-trips through the physical switch.)
- Are east-west flows subject to the same anti-spoofing, conntrack, ACL, and DPI pipeline, or a faster intra-host path? Either is defensible; pick one.

### B4. ARP / ND / DHCP gateway behavior

`DESIGN.md §2` says "ARP proxying and IPv6 ND are explicitly managed" but the docs do not specify *what* that management is. For each protocol, decide and write down:

- ARP: who answers requests for the gateway IP? For other VM IPs on the overlay? For arbitrary off-link IPs (proxy-ARP)?
- IPv6 ND: NA/NS for the gateway link-local and global addresses; RA emission policy; RA guard on the VM-facing side.
- DHCPv4 / DHCPv6: is the dataplane a DHCP server, a relay, or neither? If neither, how do VMs get addresses? DHCP guard on the VM-facing side is mentioned in codex but not in the revised docs.

### B5. Control-plane API and VM identity provisioning

`DETAILED_DESIGN.md §3.3` says identities come "from control-plane VM identity" but no API exists in the docs. Specify:

- How a VM is registered (RPC, gRPC, file, sysfs, libvirt hook, …).
- The identity record shape: VM-ID, allowed MAC(s), allowed IPv4/IPv6 (including DHCP-assigned ranges), allowed VLAN tag, queue/core assignment, tenant ID, conntrack zone, rate limits.
- Lifecycle: register, update, drain, deregister; what happens to in-flight flows on each transition.
- Authorization model for the control plane (who can call it, mTLS, capability tokens).

This is not optional — every other module references it.

### B6. vhost-user feature bits

`DESIGN.md §2` mentions "split virtqueues, multiqueue, eventfd kick/call" but does not enumerate the negotiated feature set. List the v1 feature mask, e.g.:

- Required: `VIRTIO_NET_F_MAC`, `VIRTIO_NET_F_STATUS`, `VIRTIO_NET_F_CTRL_VQ`, `VIRTIO_NET_F_MQ`, `VIRTIO_F_VERSION_1`, `VIRTIO_F_IOMMU_PLATFORM` (where applicable), `VHOST_USER_PROTOCOL_F_MQ`, `VHOST_USER_PROTOCOL_F_REPLY_ACK`, `VHOST_USER_PROTOCOL_F_CONFIG`.
- Disabled in v1: `VIRTIO_NET_F_GUEST_TSO4/6`, `VIRTIO_NET_F_HOST_TSO4/6`, `VIRTIO_NET_F_GUEST_UFO`, `VIRTIO_NET_F_MRG_RXBUF`, `VIRTIO_NET_F_GUEST_CSUM`, `VIRTIO_NET_F_CSUM` (re-enable later behind an offload-correctness test gate).
- Live migration / reconnect: stated explicitly as out of scope for v1, or defined.

### B7. AF_XDP multi-buffer and MTU

`DETAILED_DESIGN.md §2.2` says "Header rewrites recalculate … FCS, IPv4 checksums, TCP/UDP pseudo-headers" but does not commit on:

- AF_XDP multi-buffer (`XDP_USE_SG`) — required for jumbo / non-1500 MTU.
- Maximum MTU and whether jumbo frames are supported.
- UMEM chunk size and headroom (typical: 2048 with 256B headroom; multi-buffer needs 4096).

Pick: v1 single-buffer, MTU 1500, chunk 2048, headroom 256. Multi-buffer is a v2 feature.

### B8. NIC / driver / kernel matrix

Codex item 21. The rewrite added "SSE4.2/AVX2 baseline" but no NIC/driver matrix. Add a concrete table for v1 development and CI:

| Kernel | NIC | Driver | Native XDP | AF_XDP ZC | Multi-buf | Status |
|--------|-----|--------|-----------|-----------|-----------|--------|
| 6.6 LTS | Intel E810 | ice | yes | yes | yes | primary |
| 6.6 LTS | Mellanox CX-6 | mlx5 | yes | yes | yes | primary |
| 6.6 LTS | Intel X710 | i40e | yes | yes | no | secondary |
| 6.1 LTS | virtio-net (QEMU) | virtio_net | yes | copy-only | no | dev/CI only |

State that production refuses to start when AF_XDP falls back to copy mode unless explicitly permitted in config.

### B9. Drop reason enumeration

`DETAILED_DESIGN.md §6` mentions "drop reason enumerations" without enumerating them. The drop-reason table is part of the ABI between dataplane and metrics/logs. Sketch it now, even tentatively:

```
parser_l2_short, parser_l3_short, parser_l4_short, parser_options_overflow,
spoof_mac, spoof_ip, spoof_vlan, spoof_arp, spoof_nd, spoof_dhcp,
acl_deny, acl_no_match,
conntrack_no_state, conntrack_invalid_state, conntrack_table_full,
nat_port_exhausted, nat_no_mapping,
ratelimit_pps, ratelimit_bps,
ring_full_tx, ring_full_proxy, umem_exhausted,
dpi_failed_closed, dpi_overlap_strict, dpi_buffer_full,
quic_default_drop,
proxy_unavailable, proxy_timeout,
config_invalid, config_missing,
xsk_unbound, xdp_redirect_failed
```

### B10. BPF/XDP runtime contract

Codex item 13 is partly answered. Still missing:

- Map specs: `xsks_map` size = number of NIC queues; pin path; uid/gid.
- Privilege model: `CAP_BPF` + `CAP_NET_ADMIN` at startup, dropped after attach.
- Coexistence: `libxdp` dispatcher policy if other XDP programs may attach.
- Detach behavior on process exit (the program should *stay* attached and continue to fail-closed-DROP, or detach cleanly — pick one and write it down).

### B11. Slow-path threading model

`DETAILED_DESIGN.md §3.1` says "one dataplane thread per queue group" but never explains where the slow path runs. Concretely:

- Conntrack creation, NAT allocation, ACL miss, DPI buffering — same thread, separate worker pool, or per-NUMA workers?
- If same thread: state the latency budget for slow-path work and the per-iteration cap.
- If separate: define the handoff queue (SPSC ring? mpmc?), per-tenant fairness in that handoff, and how cross-core handoff is reconciled with codex item 10's "avoid cross-core handoff."

Recommend: keep ACL evaluation, conntrack creation, and NAT allocation on the dataplane thread (data is hot in cache); push only DPI full-stream and proxy-handoff to a separate worker pool, with a bounded SPSC per dataplane thread.

### B12. Backpressure on the vhost-user side

The TX-full behavior on the AF_XDP side is mentioned ("tail-drop"). The matching behavior toward the guest is not. When the dataplane cannot accept more from `Vhost_RX` (e.g., proxy worker pool full, conntrack table full for that tenant), state which of these v1 implements:

- Stop pulling from the guest virtqueue (natural backpressure via virtqueue full).
- Drain and tail-drop in userspace.
- Drop with explicit virtio-net error counter.

Recommend the first — it propagates congestion to the guest cleanly and avoids burning CPU on packets that will be dropped anyway.

### B13. Conntrack "zone" is referenced but not defined

`DETAILED_DESIGN.md §3.2` includes "conntrack zone" in the flow key but never defines what assigns the zone. Define: zone = tenant ID by default; optionally subdivided per VLAN or per VM-ID for stricter isolation. State whether NAT mappings are per-zone.

### B14. Memory budgets

Performance targets are pps-based but not memory-based. Add:

- Conntrack: bytes per flow × max flows × number of zones = bound.
- NAT table: entries × bytes.
- DPI per-flow buffer cap.
- UMEM total (frames × chunk size).
- Per-tenant share of each.

A reader should be able to compute "what does 1M concurrent flows across 100 tenants cost in RAM?"

## C. Smaller but Worth-Doing Improvements

### C1. QUIC default-drop scope

`DESIGN.md §2` says "Default DROP policy for QUIC (UDP/443)." Tighten:

- Does this drop *all* UDP/443 or only UDP/443 not matching an explicit allow rule?
- Does it cover QUIC on non-443 ports (HTTP/3 alt-svc, QUIC on 80, 8443)? Recommend: drop UDP-with-QUIC-handshake-signature regardless of port, or be honest that policy is port-based and bypassable.
- Is ICMP "fragmentation needed" required to elicit TCP fallback from clients?

### C2. CPU isolation specifics

The rewrite mentions "isolated CPU cores" but no concrete kernel boot flags. State the v1 baseline: `isolcpus=`, `nohz_full=`, `rcu_nocbs=` for the dataplane cores; IRQ affinity excluding those cores; `SCHED_FIFO` priority for dataplane threads (or a defended decision not to use RT).

### C3. Sandboxing posture

`DESIGN.md §5` says "minimal capabilities, seccomp, strict cgroups." Concretely list, even if v1 is permissive:

- Capabilities retained at runtime: e.g., `CAP_NET_RAW` (none if BPF is pre-attached), `CAP_BPF` (drop after setup), `CAP_NET_ADMIN` (drop after setup), `CAP_IPC_LOCK` for UMEM mlock.
- Seccomp: allowlist or denylist. A small allowlist is achievable for a dataplane.
- Namespace isolation: net namespace for the proxy delivery veth, mount namespace, PID namespace.
- The control plane is a separate process with separate privileges (recommend yes).

### C4. Test plan needs the codex adversarial cases

`DETAILED_DESIGN.md §6` lists three test layers and "chaos testing." Codex enumerated specific evasion test cases (TCP overlap, IPv6 ext-headers, QUIC, ARP/ND/DHCP spoofing, VLAN hopping, flow-table exhaustion, mixed-tenant TX-full). Copy that list verbatim into the test plan as required cases — they are how the security claims get verified.

### C5. "Hyperscan" is named but not committed

`DETAILED_DESIGN.md §3.4` names Hyperscan as if chosen. Hyperscan has CPU-feature constraints (codex item 20), licensing notes (BSD-3 since the Intel relicense, but the older repo was Boost), and competes with Vectorscan. Either commit ("v1 uses Vectorscan; Hyperscan optional on Intel-only deployments") or label it as "regex engine TBD; placeholder Hyperscan/Vectorscan" so reviewers do not assume the choice is locked.

### C6. Live migration / reconnect

Even if v1 says no, write that down. Otherwise QEMU users will assume yes.

### C7. Roadmap step 1 is the doc itself

Roadmap step 1 says "finalize network mode, interface mode, fail-closed policy, and offloads." With items A1–A4 unresolved, step 1 is not actually done. Either rewrite step 1 as "publish v1 ADR resolving items A1–A4 and B1–B7," or fold those resolutions into `DESIGN.md` and remove step 1.

## D. Suggested Concrete Edits

These are the smallest changes that close the worst gaps without re-architecting anything.

0. **`README.md`** — five edits to match the design docs:
   - Replace "Transparent Proxy Gateway" with whichever mode `DESIGN.md §2` actually commits to (see A2 / D2 — recommend "Routed gateway with selective proxy").
   - Replace the "preventing DMA corruption from hostile guests" bullet with: *"Strict single-copy boundary at the VM interface; guest memory is never DMA-mapped to the NIC in v1."*
   - Drop "DHCP" from the anti-spoofing bullet until DHCP guard is specified, or add DHCP guard to `DESIGN.md §3.3` and `DETAILED_DESIGN.md §3.3` first.
   - Change the AF_XDP bullet to: *"Native-driver XDP with AF_XDP zero-copy on supported NICs (E810, mlx5, i40e); copy-mode and generic XDP are CI/dev fallbacks only."*
   - Sync the README roadmap with `DESIGN.md §6` — either copy all 11 items or label the README list as a summary and link to the canonical list.

1. **`DETAILED_DESIGN.md §2.1`** — replace "Global Pool: Shared for physical NIC I/O" with: *"UMEM pools are per-NUMA-node. On a single-socket host, this collapses to one global pool. Each NIC RX/TX queue is bound to the UMEM on the queue's NUMA node."*

2. **`DESIGN.md §2 Network Mode`** — rename "Explicit proxy gateway" to "Routed gateway with selective proxy" and add: *"The system owns the gateway IP/MAC for each VM subnet. Flows are routed/NATed by default; selected flows (per ACL) are diverted to the proxy delivery path before egress."*

3. **`DESIGN.md §4 Packet Flow`** — add a fourth bullet: *"Proxied egress (selected flows): `Vhost_RX guest buffer -> UMEM frame -> policy match -> push to proxy veth in proxy netns -> proxy connects upstream -> upstream response re-enters dataplane as a fresh flow.`"*

4. **`DESIGN.md §2 Proxy Delivery Path`** — replace the "External/local proxy …" sentence with: *"v1 uses a single delivery mode: a Linux network namespace housing the proxy, connected to the dataplane via a veth pair. Original destination (pre-NAT) is conveyed to the proxy via PROXY-protocol v2 prefixed on the inbound TCP stream. Return traffic from the proxy is treated as a new flow under the same tenant identity."*

5. **`DETAILED_DESIGN.md`** — add §3.5 NAT Engine, §3.6 ARP/ND/DHCP Plane, §3.7 Control Plane API, each one paragraph + a short list; flesh out later.

6. **`DETAILED_DESIGN.md §3.4`** — change "Hyperscan scratch regions" to "regex engine scratch regions (Vectorscan in v1; Hyperscan-compatible API)."

7. **`DETAILED_DESIGN.md §6`** — append the codex adversarial test list as `§6.1 Required Adversarial Cases`.

8. **Both docs** — add an explicit "Out of Scope for v1" section listing: live migration, AF_XDP multi-buffer, jumbo frames, TLS interception, DPDK fallback, hardware XDP offload, IPv6-only deployments (or state IPv6 is in scope, but be explicit).

## E. Summary

The design is good enough to start prototyping the I/O proofs (roadmap steps 2–4) *if* the contradictions in section A — including the new README drift in A0 — are resolved first. Items in section B can be filled in while the I/O proofs are running, but they must all be answered before step 5 (parser + anti-spoofing) starts, because the anti-spoofing module reads identity records that B5 has not defined yet.

The codex review was thorough and the absorption was real. The remaining work is moving from "we acknowledged the constraint" to "we wrote the contract" — and now that there is a `README.md` acting as the front door, keeping it in lockstep with the design docs.
