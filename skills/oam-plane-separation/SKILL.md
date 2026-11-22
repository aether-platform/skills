---
name: oam-plane-separation
description: Separate the service plane and the OAM (operations, administration and maintenance) plane of a network function running in Kubernetes Pods, using Multus secondary interfaces and the CNI vrf plugin, and prove the separation with repeatable checks. Use when a Pod must carry user traffic and management traffic on different interfaces and VRFs, when you need to design the NetworkAttachmentDefinitions and socket bindings for that, when you must show that service traffic never crosses the OAM plane (and the reverse), or when isolation checks need a negative control to show they can actually detect a missing VRF.
---

# OAM plane separation

This skill gives a fixed procedure for putting a network function's traffic
planes on separate interfaces and VRFs inside a Kubernetes Pod, and for
verifying that they stay separate. It uses
[Multus CNI](https://github.com/k8snetworkplumbingwg/multus-cni), the
[CNI reference plugins](https://github.com/containernetworking/plugins)
(`macvlan`, `vrf`), [whereabouts](https://github.com/k8snetworkplumbingwg/whereabouts)
for IP addresses, and standard Linux tools inside the Pod's network namespace
(`ip`, `ss`, `tcpdump`). It does not add any tools of its own.

Background: [Linux VRF](https://docs.kernel.org/networking/vrf.html) gives
each plane its own routing table. Interfaces enslaved to a VRF device only
see the routes of that VRF, and sockets bound to an interface (or to the VRF
device) only send and receive there.

For the general cluster work around this (choosing the context, dry-run,
rollout, throwaway clusters), see the `kubernetes-safe-change` skill in this
repository.

## Planes

Name the planes before writing any manifest. The usual split is:

| Plane | Carries | Where |
|-------|---------|-------|
| **Platform** | Kubernetes itself: kubelet probes, the cluster CNI's Pod network, image pulls | `eth0`, owned by the cluster CNI. Not part of either VRF; its routes can change with the platform. |
| **Service** | The traffic the function exists for, in both directions: towards its clients (for example DNS and TLS listeners) and towards upstream servers. East-west traffic of the service (replica to replica) also stays here. | One or more secondary interfaces enslaved to the service VRF. |
| **OAM** | Management API, health and metrics endpoints for operators, log and trace export, control connections. | A secondary interface enslaved to the OAM VRF. |

- **The VRFs exchange no routes.** That is the definition of the separation:
  from the service plane you cannot reach OAM addresses and the reverse,
  except at contact points you list on purpose (for example the function's
  admin listener on the OAM address).
- **Direction inside the service plane is a socket matter, not a VRF matter.**
  Client-facing and upstream-facing interfaces can both be in the service
  VRF. The kernel answers for any of its own addresses inside the same VRF,
  so the separation between the two directions comes from each socket being
  bound to its own interface, not from routing.

## Ground rules

- **Separation is measured, not assumed.** Every claim in the report comes
  from a check in section C, with its output kept.
- **Every check needs a negative control.** Run the same checks against a
  deployment without the `vrf` plugin. A check that passes in both cases does
  not test the VRF (section D).
- **Probes are not part of the system under test.** Observation containers
  and probe Pods may need privileges (`NET_ADMIN`, `NET_RAW`) that the
  function itself must not get. Keep them separate and do not count their
  permissions in the function's security profile.
- **Lab results are lab results.** A lab built from containers as nodes and
  bridges as networks shows that the chain works and that the checks
  discriminate. Interface behaviour, CNI options and kernel permissions on the
  target platform (a managed cluster's CNI, its node kernel) must be checked
  there.
- **Use documentation address ranges in shared material.** Examples use
  192.0.2.0/24, 198.51.100.0/24 and 203.0.113.0/24. Do not publish real
  subnets, interface maps or node names.

## A. Build the chain

### A1. Install the pieces (pinned)

1. **CNI plugins** on every node, in the CNI binary directory the cluster
   actually uses (it differs between distributions; for example k3s keeps its
   own). At least `macvlan` (or `ipvlan` / `host-device`) and `vrf`.
2. **Multus**, at a fixed version. Check that its kubeconfig and CNI
   configuration paths match the distribution; with the defaults on a
   non-standard layout, every Pod fails to start.
3. **whereabouts** for cluster-wide unique addresses on the secondary
   networks. Again, set its configuration and kubeconfig paths if the
   distribution does not use the defaults.
4. Record versions and checksums of everything installed.

### A2. One NetworkAttachmentDefinition per interface

Each NAD chains the interface plugin with the `vrf` plugin. The VRF a plane
belongs to is a property of the NAD:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: nad-oam
spec:
  config: |
    {
      "cniVersion": "1.0.0",
      "name": "nad-oam",
      "plugins": [
        { "type": "macvlan", "master": "<node interface for OAM>", "mode": "bridge",
          "ipam": { "type": "whereabouts", "range": "203.0.113.0/24" } },
        { "type": "vrf", "vrfname": "oam" }
      ]
    }
```

- Write the client-facing, upstream-facing and east-west NADs the same way
  with `"vrfname": "svc"`, each on its own node interface and range.
- Generate a second set with the same names plus `-novrf`, identical except
  for the missing `vrf` step. It is the negative control.
- `master` names a node interface. If node interfaces can be renamed or
  reordered (for example when a node restarts and its networks are attached
  in a different order), the Pod lands on the wrong plane. Check the mapping
  after every node restart.

### A3. Attach the interfaces to the Pod

```yaml
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [ {"name": "nad-ue",     "interface": "net1"},
        {"name": "nad-origin", "interface": "net2"},
        {"name": "nad-oam",    "interface": "net3"},
        {"name": "nad-ew",     "interface": "net4"} ]
```

The CNI creates the VRF devices and enslaves the interfaces, so the
function's own containers do not need `NET_ADMIN` for this.

### A4. Bind every socket to its interface

- Every listener gets an explicit address and is bound to the interface of
  its plane (`SO_BINDTODEVICE`). No wildcard (`0.0.0.0` / `::`) listeners.
- Outgoing connections to upstream servers get an explicit source address and
  are bound to the upstream-facing interface.
- The admin / metrics listener is bound to the OAM address. A process that
  has sockets in both VRFs is a contact point; list it as such.
- In Envoy, for example: listener `address` plus `socket_options` with
  `SO_BINDTODEVICE` (level 1, name 25, the interface name as bytes) for
  listeners and the admin interface, and `upstream_bind_config` with a source
  address and the same socket option for clusters. Check separately where the
  application's own DNS lookups go out; they may not follow the upstream bind
  configuration.
- Check the permissions on the target kernel. Binding a socket to a device
  traditionally required `CAP_NET_RAW`; on recent kernels it can work for an
  unprivileged process. Test the function as non-root with all capabilities
  dropped first, and check its Pod spec with a server-side dry-run against
  namespaces labelled `pod-security.kubernetes.io/enforce=baseline` and
  `=restricted` (adding `NET_RAW` is rejected by both).

## B. Test fixtures

- One probe Pod per plane (client side, upstream side, OAM side), attached to
  that plane only, through the `-novrf` NADs (the probes are the outside
  world, not part of the VRF design).
- Probes need routes to specific addresses (`/32`) on their plane. Before
  every reachability check, confirm the route does not go via `eth0`: in a
  lab where nodes forward between networks, a missing route silently falls
  back to the Pod network and makes a closed path look open.
- Put probes and stand-ins on nodes that fault tests do not touch, so they
  keep their addresses.
- Use fresh request URLs per run when caches are involved, so a cached answer
  from a previous run does not hide missing upstream traffic.

## C. Checks

Run inside the function's network namespace (an observation container in the
Pod, or `kubectl debug` with a profile that allows it). Keep every output.

| # | Check | How | Expected |
|---|-------|-----|----------|
| C1 | Interface → VRF | `ip -d link show` | service interfaces have `master svc`, the OAM interface `master oam`, `eth0` no master |
| C2 | No route leak | `ip route show vrf svc`, `ip route show vrf oam`, `ip rule` | each VRF has only its own prefixes, and each VRF has at least its connected routes (an empty table passes the leak check trivially) |
| C3 | No unintended reach | from each plane's probe, to every address of the function: TCP / UDP to its ports **and ICMP** | only the listed contact points answer (client plane → client listeners, OAM plane → admin); everything across VRFs is closed, ICMP included |
| C4 | Service traffic avoids OAM | capture on the OAM interface (`tcpdump -i net3`) and compare `ip -s link` counters before and after a service test run | no packets from the test on the OAM interface (subtract the OAM probe's own traffic) |
| C5 | Direction: listeners | `ss -ltnup` | client listeners bound only to the client-facing address and interface; connecting to them from the upstream side fails |
| C6 | Direction: upstream | capture on both service interfaces during an upstream request; upstream server logs | packets to upstream only on the upstream-facing interface; the upstream sees the upstream-facing source address |
| C7 | East-west | capture during replica-to-replica traffic | only on the east-west interface, never on OAM |
| C8 | Socket ownership | `ss -tunap` per process, tabulated: process → socket → bound interface → VRF | matches the design; no unbound sockets; processes with sockets in both VRFs are exactly the listed contact points. Take it while an upstream request is in flight, or short-lived upstream sockets are missed |
| C9 | Platform plane | `ss` bindings and a capture on `eth0` during tests | the function's sockets are not bound to `eth0` and send no service traffic there |

When counting packets, match addresses as packet endpoints, not as strings
anywhere in the capture (DNS answers contain addresses too), and ignore empty
lines.

## D. Negative control

Deploy the same function with the `-novrf` NADs and run section C again.

- Expect C1, the "has its own routes" part of C2, and the ICMP cells of C3 to
  **fail**. If they pass, the check is not testing the VRF.
- Expect TCP cells of C3 and C5–C8 to still pass when sockets are bound to
  interfaces: binding alone already closes those paths. **TCP reachability by
  itself cannot show whether the VRF exists.** Keep ICMP and the direct
  observations (C1, C2) in the pass criteria.
- Inside one VRF, the function answers ICMP to its other addresses in that
  same VRF (for example upstream side → client-facing address). That is
  expected; write the expectation that way instead of "no answer".

## E. Fault cases

Repeat the relevant checks during and after:

- **Upstream-facing interface down** (`ip link set net2 down` from the
  observation container): the function fails closed — no fallback through
  the client-facing interface, `eth0` or OAM. After it comes back, the
  service answers again and section C passes again.
- **Pod deleted / node drained / node paused**: surviving replicas keep
  answering, nothing reroutes through OAM, replacements come up with the same
  plane mapping (re-run C1–C3 on them).
- Fault-injection tools that create long-lived objects (for example a chaos
  experiment resource) must be removed after their own step, or they disturb
  later steps. Check for leftovers before trusting later results.

## Report

```
Planes:      platform=eth0  service VRF=<name>: <interfaces>  OAM VRF=<name>: <interfaces>
Versions:    Multus <v>  CNI plugins <v>  whereabouts <v>  function image <digest>
Privileges:  function: <user, capabilities>  PSA levels accepted: <baseline/restricted>
Checks:      C1–C9 <pass/fail per item>  reachability matrix <n/n>
Control:     -novrf: failed as expected <items>  still passing <items, why>
Faults:      <case: result>
Contact points: <process: VRFs and ports>
Not verified on target platform: <CNI, kernel, permissions>
Evidence:    <folder with outputs and captures>
```
