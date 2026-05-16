# A Kubernetes UDP Bug That's Invisible to Every Layer

I spent the better part of a week last month chasing a Kubernetes networking failure where UDP traffic clearly arrived at the node - I could see every packet in `tcpdump` - but somehow never reached the backend pod. No firewall drops anywhere. The pod was healthy and listening on its target port. The iptables NAT rule for the NodePort was in place. And yet that rule's packet counter wasn't moving despite continuous traffic on the wire.

It's the kind of failure that *sounds* impossible: the packets are there, the rules are there, the listener is there. Something has to be wrong with one of them. But the actual bug lives in none of the obvious places. It lives in a small interaction between iptables NAT, the kernel's connection-tracking subsystem, and the way Kubernetes's kube-proxy maintains iptables rules - a seam that no single component is responsible for monitoring.

This article walks through what I found, why it happens, and the four ways I know to fix it. None of the mechanism is novel - most of it is documented in pieces across kube-proxy source, Linux kernel docs, and various CNI plugin pages. What's harder to find is the *combination* described end-to-end, with the actual diagnostic signature and the operational consequences spelled out. So I wrote it down.

## The symptom, and the things that weren't broken

The affected service was a long-lived UDP listener running as a DaemonSet, behind a NodePort, with `externalTrafficPolicy: Local` so that source IPs were preserved for downstream processing. An external exporter was streaming continuously toward `node-IP:31234`. Symptoms were:

- **`tcpdump` on the node's ingress interface**: packets arriving normally, source IP and destination IP both correct, destination port 31234/UDP.
- **iptables filter-table DROP counters**: zero across the board (Calico, kube-system chains, default policies). Nothing in the firewall path was dropping the traffic.
- **Kernel reverse-path filter**: `nstat` showed `IPReversePathFilter = 0`. Strict RPF was on for the ingress NIC but it wasn't dropping anything.
- **iptables NAT rule for the NodePort**: present, correctly pointing at the right SEP chain, which correctly DNAT'd to the pod's IP and target port. Counter at zero despite continuous traffic. That last bit is the strangest part.
- **The pod**: healthy, ready, listening on the target UDP port. Pod-side `tcpdump` confirmed it was receiving nothing.
- **No NetworkPolicy** restricting ingress on the relevant port.

In other words: every component that *could* drop or misroute the packet was confirmed to not be doing so, and yet the packet was getting lost between the wire and the pod. There was a fourth-floor-elevator-to-third-floor problem somewhere, and none of the floor displays were lighting up.

## Where the truth was hiding

The answer was in conntrack.

Every UDP flow that passes through a Linux netfilter-managed host gets a connection-tracking entry. The entry has two tuples:

- **Original tuple**: the 5-tuple of the packet as it arrived (`src=<exporter> dst=<node> sport=X dport=31234`).
- **Reply tuple**: what the kernel expects packets going *back the other way* to look like. This is where the NAT translation gets recorded.

For a healthy NodePort flow that's been DNAT'd to a pod, the reply tuple shows the pod's IP and target port - that's the kernel's note to itself: "if a reply comes from the pod, rewrite it back to the original destination so the sender sees it as coming from the NodePort." A working entry looks like this:

```
udp  17 29  src=<exporter-IP>  dst=<node-IP>     sport=X        dport=31234  [UNREPLIED]
            src=<pod-IP>       dst=<exporter-IP> sport=<pod port> dport=X
```

The broken entry I was looking at had something different in the reply tuple:

```
udp  17 29  src=<exporter-IP>  dst=<node-IP>     sport=X        dport=31234  [UNREPLIED]
            src=<node-IP>      dst=<exporter-IP> sport=31234    dport=X
```

The reply tuple is just the original reversed. No DNAT in there. The kernel has recorded "this conversation stays as-is, no translation," even though the iptables NAT rule explicitly says traffic to port 31234 should be DNAT'd to the pod.

The `[UNREPLIED]` flag, incidentally, is not a sign of failure here. Fire-and-forget UDP listeners don't reply, so healthy entries stay `[UNREPLIED]` indefinitely. The smoking gun is the missing pod IP in the reply tuple, not the flag.

## How a broken entry gets created

To understand why this happens, you need three properties of how Linux netfilter handles UDP traffic. None of them is a bug. They're all sensible design decisions in isolation. The bug is the combination.

**Property 1: NAT rules are consulted exactly once per flow.** When a packet arrives that doesn't match an existing conntrack entry, the kernel walks the iptables NAT chains, applies whatever DNAT or SNAT they specify, and records the result in a new conntrack entry. Every subsequent packet of the same flow finds the entry by 5-tuple match and uses the cached translation directly. NAT chains are *never* re-evaluated for an established flow.

This is the optimization that makes Linux NAT fast. It would be prohibitively expensive to walk the chain list on every packet of every flow. So the kernel decides once, caches the answer, and trusts the cache.

**Property 2: UDP "flows" have no natural termination signal.** TCP has FIN, RST, and timeouts driven by the protocol state machine. UDP has none of that. A UDP flow is just "two endpoints sending datagrams to each other," and conntrack tracks it via an idle timer: if no packets match the entry for `nf_conntrack_udp_timeout` seconds (default 30), the entry is deleted.

This works fine for short-lived UDP exchanges like DNS queries. But a long-lived UDP exporter - NetFlow, sFlow, IPFIX, syslog, RTP, GTP-U - sends continuously from a fixed source port. Every packet matches the same conntrack entry by 5-tuple and resets the idle timer. The entry refreshes forever and never times out.

The fixed source port is load-bearing here. A naive `nc -u` loop without `-p` gets a fresh ephemeral source port on every packet, so every packet creates its own conntrack entry - broken entries can still form during the empty-rules window, but they each age out individually within `nf_conntrack_udp_timeout` and subsequent ephemeral-port flows traverse NAT cleanly. The "stuck forever" property of the bug specifically requires a single long-lived 4-tuple, which is what real-world telemetry exporters produce and what a synthetic reproducer needs to mimic if you want to study the persistence aspect.

**Property 3: kube-proxy maintains iptables rules asynchronously, with windows of inconsistency.** kube-proxy watches the Kubernetes API for Service and EndpointSlice changes, computes the iptables rule set it wants to install, and applies it via `iptables-restore`. The reconcile loop is fast in steady state but it has finite-duration windows where the kernel's rules don't reflect the current desired state - most notably during kube-proxy's own startup, while it's syncing API caches, or if the API server is temporarily unreachable.

If kube-proxy is in such a window for ~80 seconds at startup, and your UDP exporter sends a packet during that window, here is what happens:

1. The packet arrives. No conntrack entry exists for this flow.
2. The kernel walks iptables NAT. Either kube-proxy hasn't installed its DNAT rule yet, or it has but the rule is missing the SEP chain that DNAT's to the pod. Either way, no rule matches.
3. NAT chains finish without rewriting the packet.
4. The kernel records the result in a new conntrack entry: reply tuple = original reversed, no DNAT applied.
5. The packet continues with destination still `node-IP:31234`. Since no userspace process is listening on UDP 31234 in the host's network namespace, the packet is dropped (with or without ICMP port-unreachable depending on host config).
6. The conntrack entry persists. The exporter keeps sending. Every subsequent packet matches the entry, gets routed per the cached (no-)mapping, and ends up at the host port that has no listener.
7. kube-proxy finishes its sync seconds later. The iptables rule is now correct. But the kernel reads conntrack, not iptables, for established flows. **The traffic silently evaporates inside the host.**

The mental model I use for this: NAT chains are a wall of redirect rules; conntrack is a stack of sticky notes the kernel keeps next to the wall. When a new conversation arrives, the kernel reads the wall and writes a sticky note. From then on, it reads only the sticky note. The bug is what happens when the kernel writes a sticky note while the wall is briefly empty: the note says "deliver as-is," and that note now overrides any rule that gets stuck on the wall later.

## Catching it in the act

I went back later and rigged a lab cluster to reproduce the bug deterministically - a fresh boot with the exporter sending the whole time, plus a small readiness-probe delay on the listener pod so the empty-rule window was wide enough to capture. Sampling `conntrack` and `iptables` every half-second across the post-boot recovery window gave me the moment the bug forms in single-snapshot resolution. Two consecutive snapshots, ~500 ms apart, tell the whole story.

Just before kube-proxy's first sync finishes (the "wall empty" state):

```
udp  17 29  src=<exporter-IP>  dst=<node-IP>     sport=X        dport=31234  [UNREPLIED]
            src=<node-IP>      dst=<exporter-IP> sport=31234    dport=X
```

(No `KUBE-NODEPORTS` rule installed yet; SVL chain not present. Reply tuple is just the original reversed - no DNAT.)

500 ms later, after kube-proxy's first iptables-restore lands:

```
udp  17 29  src=<exporter-IP>  dst=<node-IP>     sport=X        dport=31234  [UNREPLIED]
            src=<pod-IP>       dst=<exporter-IP> sport=<pod port> dport=X
```

(NAT rules now present; reply tuple now shows the pod's IP and target port. Same 4-tuple as before; healthy DNAT.)

A conntrack entry's reply tuple is immutable once set - the kernel doesn't edit existing entries, only creates and deletes them. So the apparent "flip" between these two snapshots is necessarily an entry deletion plus a recreation: kube-proxy's reconcile path explicitly flushed the broken entry as part of installing its rules, and the next exporter packet (within the same 500 ms window) created a fresh entry that hit the just-installed DNAT. This turns out to matter - it's what newer versions of kube-proxy now do reliably, and it's the basis for the "fifth thing" further down.

## Why it's hard to notice

The really evil part of this failure mode is that **no counter moves**:

- Filter-table DROP counters: zero. Nothing dropped the packet on a policy basis.
- NAT-table counters: zero. NAT was never consulted (conntrack made the decision).
- Kernel RPF counters: zero. Routing was fine.
- The packet is visible on the wire and in `tcpdump -i any` because tcpdump taps the network stack above the conntrack/NAT path. So it doesn't even look "lost."

The traffic just quietly disappears between layers, and the only place the truth is visible is in the conntrack table itself - which most operators don't routinely check for the specific reply-tuple field that distinguishes a healthy entry from a broken one. Downstream consumers notice missing data, source teams notice their reports look thin, and the usual triage path (network drops? firewall? source side?) returns nothing actionable. I've seen this take days to diagnose more than once.

## Four ways to fix it

The fixes form a hierarchy, from "patch one stuck flow on one node" to "make the bug class structurally impossible."

### Per-flow remediation: `conntrack -D`

The one-line incident fix:

```bash
sudo conntrack -D -p udp --dport <NodePort>
```

This tears up the sticky note. The very next packet of the flow has no cached entry to follow, so the kernel walks the NAT chains again. Provided kube-proxy's rules are now in place (which they typically are by the time anyone notices the symptom), a fresh, healthy conntrack entry is created with proper DNAT. Recovery is on the order of seconds.

A caveat: it matters that the iptables rules are correct at the moment the next packet arrives. If something external has flushed kube-proxy's chains (more on this below) and the rules are still empty, `conntrack -D` will just produce another no-DNAT entry within milliseconds. In normal operation, where kube-proxy's rules are intact, this is the cleanest possible fix.

On newer kube-proxy versions, the caveat is largely self-resolving: kube-proxy's reconcile path flushes conntrack itself when it installs rules, so the order of operations matters less. On older kube-proxy versions, the caveat is real and worth checking the rules before flushing the entry.

### Remote remediation: trigger a kube-proxy resync via a real Service change

When you don't have shell access to the affected node, you can force kube-proxy to flush conntrack on your behalf - but it's surprisingly picky about which signals will do it.

`helm upgrade` of the affected workload works (verified empirically). The combination of a pod rollout plus the Service spec being re-applied produces enough Service/Endpoint state change that kube-proxy fires its `ClearEntriesForPort` cleanup, which flushes conntrack entries by destination port - including the broken one.

What does **not** work, based on testing: `kubectl annotate svc/endpointslice`, `kubectl delete svc` (deleting and recreating the Service is not enough to trigger the relevant cleanup path on the kube-proxy build I tested), and `kubectl delete pod -n kube-system kube-proxy-<node>` (the iptables rules persist in the kernel across kube-proxy process restarts, and the new instance treats existing chains as already-managed).

If you need to do this remotely, prefer Helm or any other path that produces a real Service or EndpointSlice content change. Annotation-only events are filtered out by kube-proxy's change tracker.

One caveat worth flagging, because I tripped over it later: the "what does not work" list above is what I observed on the kube-proxy version originally affected. When I went back and retested on a newer cluster, `kubectl annotate svc/<name>` *did* heal the symptom - the newer kube-proxy's reconcile path appears to do more work than the older one, including a `ClearEntriesForPort`-style conntrack flush on every reconcile (not just on real-content changes). So if you're on a recent enough kube-proxy, the simplest remote remediation may actually be a no-op annotation. On the older build I tested originally, it was reliably a no-op in the unhelpful sense. Test against your own version before relying on it.

### Per-Service structural fix: `hostNetwork: true`

If a specific Service is hitting this bug and you can afford the trade-offs, putting its pod on `hostNetwork: true` removes it from the kube-proxy data path entirely. The pod binds in the host's network namespace, the exporter sends to `node-IP:<port>` (the application's own port, no NodePort indirection), and the pod's socket receives the packet directly. No DNAT happens because no DNAT is required.

The bug class cannot form for this Service because the mechanism that produces it isn't in the path.

This fits a specific shape of workload unusually well: DaemonSets that need exactly one pod per node, where source-IP preservation is a hard requirement (so you were going to use `externalTrafficPolicy: Local` anyway), where there's no inter-pod load balancing to lose, and where Service-mesh or NetworkPolicy enforcement doesn't depend on the pod being in the pod-network namespace. Long-lived UDP listeners with per-node deployment fit this exactly.

Trade-offs: only one pod per node can listen on a given port (already implicit in DaemonSet-per-node deployment), reduced isolation from the host network namespace, NetworkPolicy applicability shifts to host-level firewalling, and the Service object becomes vestigial for these pods. For typical UDP-listener use cases none of these are showstoppers; for other workload shapes (replicated Deployments, anything that needs Service-mesh integration), `hostNetwork` is the wrong tool.

### Cluster-wide structural fix: `--proxy-mode=ipvs`

Switching kube-proxy to IPVS mode (`--proxy-mode=ipvs`) eliminates the failure class globally. I tested this in a lab cluster - same exporter pattern, same induction conditions that produce a stuck no-DNAT entry in iptables mode - and observed no stuck entry created at all.

Two mechanisms combine:

**No fall-through path.** IPVS handles load balancing in the kernel via its own connection table. When an IPVS virtual server has no real-server members (the equivalent state to an empty `KUBE-SVL-…` chain in iptables mode), packets to it are dropped at the IPVS layer - they don't continue through with no DNAT applied. The kernel never creates a conntrack entry pointing at a non-existent destination, because the packet never makes it past IPVS in the first place.

**`net.ipv4.vs.expire_nodest_conn=1`** - a kube-proxy default in IPVS mode - proactively expires existing conntrack entries when their destination real-server disappears. So even the "endpoint went away" subcase of the bug is structurally prevented: stuck entries pointing at gone endpoints don't accumulate.

In the lab test, after I removed the only real server from the IPVS virtual server, `conntrack -L` returned empty across a 15-second window with the exporter actively sending. No entry was created. When kube-proxy reconciled and restored the real server (9 seconds later, on this build - faster than the iptables Proxier's recovery from a similar disturbance), a healthy entry formed naturally on the next packet.

Trade-offs: it's a cluster-wide proxy-mode migration affecting every Service. IPVS has its own tunables (`conn_reuse_mode`, `expire_nodest_conn`, `expire_quiescent_template`) that you should review before going to prod. Diagnostic tooling shifts from iptables-introspection to `ipvsadm -L -n` and `ipvsadm -L --connection`. Some Kubernetes distributions default to iptables proxier and may have slightly less battle-tested IPVS paths.

Comparing the two structural fixes:

| Property | `hostNetwork: true` | `--proxy-mode=ipvs` |
| --- | --- | --- |
| Scope | Per Service | Cluster-wide |
| Effort | Pod-spec change, redeploy | Cluster-wide proxy-mode migration |
| Risk | Low; only affects pods you change | Medium; touches every Service's data path |
| Reversibility | Remove `hostNetwork: true` | Switch proxy-mode back |
| Source-IP preservation | Free (no `externalTrafficPolicy` needed) | Still needs `externalTrafficPolicy: Local` |
| Service-mesh / NetworkPolicy applicability | Reduced for affected pods | Unchanged |
| Fixes the bug class for other Services | No | Yes |
| Operational familiarity | Standard K8s feature | Off-default for many distros |

For a single Service hit by this bug whose workload shape fits, `hostNetwork: true` is often the smaller, cleaner fix. For a cluster where the bug pattern could plausibly affect future Services too, the cluster-wide IPVS migration is the more comprehensive answer. They're not mutually exclusive.

## A fifth thing worth knowing: kube-proxy version matters

This isn't a fifth fix; it's an observation about the threat model that I didn't fully appreciate until I went back and retested. The kube-proxy version on your cluster reshapes the danger of this bug substantially.

On the older kube-proxy build originally affected, a broken conntrack entry is essentially permanent once created. Nothing routine clears it. `conntrack -D` or `helm upgrade` works (those are deliberate operator actions); pod restarts don't, annotation pokes don't, and kube-proxy's own periodic sync doesn't, because conntrack isn't part of kube-proxy's reconcile model on that build. The bug is sticky, and the only way out is the human one-liner.

On newer kube-proxy builds, broken entries still form by the same mechanism - I reproduced one cleanly in the lab (the snapshot pair from earlier in this article is from that exact run). What changed is that kube-proxy now flushes conntrack for the affected port as part of essentially every reconcile triggered by Service or EndpointSlice state changes. The mechanism is the same `ClearEntriesForPort`-style flush that `helm upgrade` already used to lean on, except now it's wired into a much wider range of triggers. Pod restart, helm upgrade, annotation, even the routine pod-readiness transition that follows a fresh boot - all of them clear the broken entry within seconds.

A short comparison:

```
                                       Older kube-proxy   Newer kube-proxy
Bug forms during a kube-proxy gap      yes                yes
`conntrack -D` heals                   ✓                  ✓
`helm upgrade` heals                   ✓                  ✓
`kubectl annotate svc/endpointslice`   no-op              heals
Pod restart / readiness change heals   no                 yes
First-sync at kube-proxy startup       n/a                heals on boot
```

The bug *can* still persist on a newer kube-proxy build, but only when kube-proxy itself is unable to reconcile - for instance, when it's lost API-server connectivity for the duration the broken entry needs to survive, which is exactly the original incident pattern. In other words: on newer clusters, the bug class is largely self-healing, and the residual risk narrows to "kube-proxy can't reconcile," which is closer to a kube-proxy-availability problem than a conntrack problem.

If you're operating a mixed fleet across kube-proxy versions, this distinction matters for triage. Same symptom, same diagnostic signature, but very different stickiness.

## A footnote: kube-proxy doesn't auto-repair externally-flushed chains

While reproducing this bug on a test cluster, I noticed a sub-finding worth flagging.

I induced the bug by externally flushing a kube-proxy-managed iptables chain (`iptables -t nat -F KUBE-SVL-<hash>`), then waited to see how kube-proxy would respond. The expectation was that kube-proxy would notice its rules had drifted and reinstall them within the default 30-second sync interval.

It didn't. Across a 90-second window - and even after a `kubectl delete pod` of kube-proxy itself, which I expected to force a clean re-sync - the chain stayed empty. Annotating the Service and EndpointSlice didn't trigger a re-sync either; those annotation events get filtered by kube-proxy's `serviceChangeTracker` because annotations don't change any field kube-proxy uses for rule generation.

The reason traces back to two kube-proxy design choices:

1. **`iptables-restore --noflush`**. kube-proxy applies its rules with this flag, and under `--noflush` the chain-declaration syntax at the top of the buffer (`:KUBE-SVL-… - [0:0]`) does *not* flush an existing chain - it only creates it if absent. So even when a sync runs, it appends rules to whatever the chain already contains rather than replacing the chain's contents.

2. **"Skip if buffer unchanged" optimization**. kube-proxy caches the last-successfully-applied buffer and skips the `iptables-restore` call entirely if the new buffer is identical. This is a real performance optimization - applying the full ruleset on every sync is expensive on large clusters - but it means kube-proxy doesn't compare its internal model against actual kernel state. It only knows what it *thinks* it wrote.

Combined: when something externally flushes one of kube-proxy's managed chains, kube-proxy's internal model still says "this chain has the right rules in it." Its next sync produces a buffer identical to the cached one, so the iptables-restore call is suppressed entirely. The chain stays empty.

Practical consequence: any third-party tool that touches iptables on a kube-proxy-managed node - security agents, observability sidecars, ad-hoc admin actions - has the ability to silently break Service traffic until either (a) the underlying Service or EndpointSlice changes in a meaningful way, or (b) someone manually restores the rule. The `kubectl annotate` resync trick documented in some operational playbooks does *not* work for this state.

This isn't strictly part of the UDP bug, but it's adjacent to it and worth knowing if you're in this layer.

A version-aware note (because I retested this later, same as I did with the remote-remediation list above): the no-op annotation behavior is what I observed on the older kube-proxy build. On newer kube-proxy versions, the same annotation does trigger a real reconcile that re-installs the externally-flushed chain and flushes conntrack for the affected port - i.e. the failure mode this footnote describes is structurally less reachable on newer builds. The `iptables-restore --noflush` and "skip if buffer unchanged" mechanisms above are still real and still describe what kube-proxy does internally; what's changed is that newer builds appear to do more work outside that fast path on Service-related events, which is enough to recover the kernel state in cases where the older build would have stayed stuck.

## The bug lives in the seam between layers

What I keep coming back to: nothing here is broken.

kube-proxy correctly maintains its iptables rules based on the current Service and EndpointSlice state. iptables correctly applies whatever rules are present. conntrack correctly caches the NAT decision that was made at flow setup, and correctly applies that cached decision to every subsequent packet of the flow. The kernel correctly hands packets between these layers per their documented contracts. Every component, looked at on its own, is doing exactly the right thing.

The failure is emergent across the seam between them - the space that no single component is responsible for monitoring. kube-proxy doesn't watch conntrack. The kernel doesn't compare conntrack against iptables. iptables doesn't notice when conntrack starts disagreeing with it. Each layer is a correct implementation of its own contract, and the bug lives in the implicit assumption that the layers will stay consistent over time, an assumption that the systems don't actively enforce.

This is why the bug takes days to find. The path you'd usually take - "where is the packet getting dropped?" - returns nothing, because the packet isn't being *dropped*. It's being correctly delivered to the address that conntrack believes is the correct destination, which happens to be a host port with no listener. Every layer would, if asked, confirm that it did its job. Nothing has anyone to point at.

The fix at incident-time is `conntrack -D` and it takes a second. But the bug class is the kind that gets baked into systems when correct-in-isolation pieces are composed without an end-to-end consistency invariant. We have a lot of these in our infrastructure, and most of them are silent until they're not. The interesting question, the one I'm still chasing, isn't "how do we fix this specific bug" but "how do we get better at finding the next one of these before it costs us a week."

One small reason for optimism: the seam isn't immutable. Newer kube-proxy builds appear to be starting to police the boundary they used to ignore - flushing conntrack as part of their reconcile path, so that even when the broken sticky note gets written, it gets torn up the next time anything Service-related happens. The fundamental seam-bug shape is still there (the broken entry can still form), but the window of consequence is now seconds rather than indefinite. That's not a fix for the class of bug in general; it's one specific seam where the layers are starting to talk to each other a little more. The interesting work, presumably, is in noticing the next seam *before* a 1.x.x release notices it for you.

If anyone reading this has worked through similar seam-between-layers bugs in their own systems - particularly the ones where every layer's diagnostics come back clean - I'd be curious to compare notes.
