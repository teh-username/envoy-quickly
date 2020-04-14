# Envoy, quickly!

A PoC on Envoy sidecars without a control plane for fast observability. Tested with [Envoy v1.14.1](https://github.com/envoyproxy/envoy/releases/tag/v1.14.1) and [k3s v1.17.4+k3s1](https://github.com/rancher/k3s/releases/tag/v1.17.4%2Bk3s1)

## Why

You are asked to setup dashboards for backend observability (golden signals, etc.) for the first time with the following constraints:

0) You've just started on the job a few weeks before
1) It should be up and running in a few days
2) Backend is composed of microservices running in Kubernetes
3) You can contribute to backend source code but it takes weeks before it gets released
4) Data should stay inside the same datacenter where the backend is deployed
5) SaaS is okay but (see 1 and 4)
6) No prod-like environment to test on so you can't build anything too elaborate
7) Can't use any service meshes (Istio, Linkerd) (see 6)

How?

## Solution

We deploy Envoy as a sidecar but configured only to be a forward proxy. We retain the business-as-usual flow of traffic by not doing anything fancy but we get metrics out of thin air.

Bonus feature: Tracing can also be achieved provided our application isn't too aggresive with dropping unfamiliar headers.

## Code Organization

The PoC is built in multiple stages, with each being a standalone set to verify whether particular features are feasible.
