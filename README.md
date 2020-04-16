# Envoy, quickly!

![Tragedy of Darth Plagueis the Wise](stage-4/demo.gif)

A PoC on injecting Envoy sidecars without a control plane for fast observability. Tested with [Envoy v1.14.1](https://github.com/envoyproxy/envoy/releases/tag/v1.14.1) and [k3s v1.17.4+k3s1](https://github.com/rancher/k3s/releases/tag/v1.17.4%2Bk3s1)

See how it was built: [stage 1](./stage-1), [stage 2](./stage-2), [stage 3](./stage-3)

See it in action: [stage 4](./stage-4)

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

We use Envoy as a [forward proxy](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_proxy#arch-overview-http-dynamic-forward-proxy). Ingress and egress will still be routed to Envoy (via iptables) but there's no need to update the config should a new service be introduced.

Bonus feature: Tracing can also be achieved provided our application isn't too aggresive with dropping unfamiliar headers.
