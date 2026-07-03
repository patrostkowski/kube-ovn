# Kube-OVN SwitchLBRule Health Check: Stuck IPPortMapping for Selector-less, Secondary-Interface Backends

## Background

`my-controller`'s `TCPVipReconciler`/`TCPEndpointSliceReconciler`
(`pkg/controllers/tcp_controller.go`, `pkg/controllers/endpointslice_controller.go`)
expose each Kamaji `TenantControlPlane`'s API server via a kube-ovn
`SwitchLBRule` VIP addressed from RFC 6598 shared space (`100.64.0.0/10`),
deliberately outside every tenant subnet's CIDR. The backend is the TCP
pod's **secondary** OVN interface (`eth1`, a Multus-attached NAD) — the pod's
primary interface (`eth0`) is the mgmt cluster's Flannel network.

`SwitchLBRule.spec.endpoints` is populated directly (no `spec.selector`) from
a `discovery.k8s.io/v1 EndpointSlice` that `TCPEndpointSliceReconciler`
maintains, itself filtered to only `Ready`-conditioned pods. This matters for
the root cause below: a `SwitchLBRule` backed by `spec.endpoints` produces a
**selector-less** backing `Service`.

## The Problem

A worker VM on the tenant subnet cannot reach the VIP — `nc`/`curl` just
hangs and times out. Every Kubernetes-level object looks correct:

```
$ kubectl get switch-lb-rules.kubeovn.io gamma-gamma-zhhct-tcp-vip -o yaml
spec:
  endpoints:
  - 10.103.0.6
  namespace: gamma
  ports:
  - {name: apiserver, port: 6443, protocol: TCP, targetPort: 6443}
  - {name: konnectivity, port: 8132, protocol: TCP, targetPort: 8132}
  sessionAffinity: ClientIP
  vip: 100.96.0.10

$ kubectl get endpointslices.discovery.k8s.io -n gamma gamma-tcp-vip -o yaml
endpoints:
- addresses: [10.103.0.6]
  conditions: {ready: true}
```

Right VIP, right endpoint, `ready: true`. The backend pod itself is healthy
(`curl localhost:6443` from inside it returns `403`, a normal unauthenticated
apiserver response). Nothing here explains the timeout.

## Root Cause

`ovn-trace --ct=new` from the worker's logical port to the VIP shows the OVN
dataplane's actual decision:

```
13. ls_in_lb (northd.c:8655): ct.new && ip4.dst == 100.96.0.10 && tcp.dst == 6443, priority 120, uuid 9d56b5d7
    drop;
```

Not `ct_lb_mark(backends=...)` — an explicit `drop`. OVN's own load balancer
believes there is no healthy backend for this VIP:port, despite the
Kubernetes-level config being entirely correct.

Cross-checking the NB/SB health-check state confirms it:

```
$ ovn-nbctl --columns=name,vips,health_check find load_balancer name=vpc-gamma-tcp-sess-load
name         : vpc-gamma-tcp-sess-load
vips         : {"100.96.0.10:6443"="10.103.0.6:6443", "100.96.0.10:8132"="10.103.0.6:8132"}
health_check : [68cdf245-f336-4d29-a1f4-621ed90b0b79, a50679f3-5131-4653-802f-30734a28cf61]

$ ovn-nbctl list Load_Balancer_Health_Check
_uuid        : 68cdf245-f336-4d29-a1f4-621ed90b0b79
external_ids: {switch_lb_subnet=gamma-gamma-primary}
options      : {failure_count="3", interval="5", success_count="3", timeout="20"}
vip          : "100.96.0.10:8132"
... (second entry for :6443, same shape)

$ ovn-sbctl list Service_Monitor
(empty)
```

The NB side is fully configured — correct VIP mapping, a `Load_Balancer_Health_Check`
row per port with sane options, and kube-ovn's auto-created reusable
health-check source VIP exists too (`kubectl get vip` shows a `gamma-gamma-primary`
entry at `10.103.0.7`, kube-ovn's per-subnet monitor source address). But the
SB side — where `ovn-controller` on a chassis actually turns this into a real
probe — has **nothing**: `Service_Monitor` is completely empty. Nothing is
being monitored, so OVN has no evidence the backend is healthy, and
`ls_in_lb` falls back to `drop`.

### Why: traced into kube-ovn's `pkg/controller/endpoint_slice.go`

`getIPPortMapping` picks its computation strategy based on whether the
backing `Service` has a selector:

```go
func (c *Controller) getIPPortMapping(endpointSlices []*discoveryv1.EndpointSlice, service *v1.Service, checkVip string) (IPPortMapping, error) {
	if serviceHasSelector(service) {
		return c.getIPPortMappingWithTargets(endpointSlices, checkVip), nil
	}
	// selector-less: scan every pod in the namespace by IP instead
	pods, err := c.podsLister.Pods(service.Namespace).List(labels.Everything())
	...
	return c.getIPPortMappingWithNoTargets(endpointSlices, pods, checkVip), nil
}
```

A `SwitchLBRule` backed by `spec.endpoints` (ours) produces a selector-less
`Service`, so it takes the `getIPPortMappingWithNoTargets` path: for every pod
in the namespace, `getEndpointProvider` → `getMatchingProviderForAddress`
checks each of that pod's kube-ovn "providers" (i.e. every NAD/subnet
attachment it has, primary or secondary) and matches the `EndpointSlice`
address against `pod.Annotations["<provider>.ovn.kubernetes.io/ip_address"]`:

```go
func getMatchingProviderForAddress(pod *v1.Pod, providers []string, address string) string {
	for _, provider := range providers {
		ipsForProvider, exists := pod.Annotations[fmt.Sprintf(util.IPAddressAnnotationTemplate, provider)]
		if !exists {
			continue
		}
		if slices.Contains(strings.Split(ipsForProvider, ","), address) {
			return provider
		}
	}
	return ""
}
```

This logic **is** provider-aware — it checks every network attachment a pod
has, not just its primary interface, so "kube-ovn only health-checks primary
interfaces" was ruled out as the explanation.

What actually looks broken: this computation runs **once**, at whatever
moment kube-ovn's controller first processes the `EndpointSlice` event for
this `Service`. If the backend pod's `<provider>.ovn.kubernetes.io/ip_address`
annotation isn't populated on the live `Pod` object *yet* at that exact
instant — plausible on a freshly created multi-NIC pod, where kube-ovn's own
secondary-CNI annotation write can lag behind Multus/primary-CNI setup
completing — the match fails, `IPPortMapping` comes back empty, and
`LoadBalancerAddHealthCheck` is called anyway, creating the NB-side
`Load_Balancer_Health_Check` row with nothing valid to monitor. Whatever
downstream mechanism is supposed to populate `Service_Monitor` from that
mapping has nothing to populate it with.

Confirmed this does not self-heal: waited 8+ seconds (past one full
`interval=5s` health-check cycle) with `Service_Monitor` still completely
empty. This looks like a one-shot computation with no retry on subsequent
`Pod`/annotation updates, not a timing issue that resolves on its own.

## Debugging Path (abbreviated)

1. `kubectl get switchlbrule/endpointslice -o yaml` — Kubernetes-level config
   all correct (VIP, endpoint IP, `ready: true`).
2. `curl localhost:6443` from inside the backend pod — `403`, backend itself
   healthy.
3. `ovn-trace --ct=new <switch> '...ip4.dst==<vip>&&tcp.dst==6443...'` —
   pipeline hits `ls_in_lb` and explicitly `drop`s instead of `ct_lb_mark`.
4. `ovn-nbctl find load_balancer ...` — `vips` mapping correct,
   `health_check` UUIDs present.
5. `ovn-nbctl list Load_Balancer_Health_Check` — options look sane
   (`interval=5`, `timeout=20`, etc.), one row per port.
6. `ovn-sbctl list Service_Monitor` — **empty**. This is the actual anomaly:
   NB says "monitor this," nothing on the SB side is actually monitoring
   anything.
7. `kubectl get vip` — kube-ovn's own reusable per-subnet health-check
   source VIP exists (`gamma-gamma-primary` → `10.103.0.7`), ruling out "the
   monitor has no source address" as the cause.
8. Manually set `ovn.kubernetes.io/service_health_check: "false"` on the live
   `SwitchLBRule` via `kubectl annotate` — propagated correctly to the
   backing `Service` (`ovn.kubernetes.io/service_health_check: "false"`
   visible on `kubectl get svc ... -o yaml`), but the *already-created*
   `Load_Balancer_Health_Check` rows were not retroactively removed —
   confirms the fix must be present at `SwitchLBRule` creation time, not
   applied after the fact.
9. Deleted and recreated the `SwitchLBRule` with the annotation present from
   the start → `health_check: []` on the `Load_Balancer` immediately, and
   `ovn-trace` now shows `ct_lb_mark(backends=10.103.0.6:6443; ...)` instead
   of `drop`. Confirmed fixed.


## Residual / Upstream

The underlying kube-ovn race — `getIPPortMappingWithNoTargets` computing an
empty mapping on a timing loss against the backend pod's own annotation
write, with no observed retry — is unresolved upstream. Worth reporting
against kube-ovn (see `ovn-multitenancy-issues/kubeovn-issue.md` in this repo
for the reporting format used for a related `SwitchLBRule`/`Vip` bug) for any
future case where OVN-level health-checking of a selector-less,
secondary-interface `SwitchLBRule` backend is actually wanted.
