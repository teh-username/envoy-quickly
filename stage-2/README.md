## Proxying egress traffic

_Note: From this stage onwards, we'll be deploying all our experiments in a Kubernetes cluster._

Let's attach Envoy as a sidecar to our 2 services, but proxying only egress traffic for now:

```
                    +-----------------------------------
                    |                                  |
                    |                                  v
+------------------------+            +-----------------------+
|+---------+     +-----+ |            |+-----+     +---------+|
||service-a|---->|proxy| |            ||proxy|<----|service-b||
|+---------+     +-----+ |            |+-----+     +---------+|
+------------------------+            +-----------------------+
     ^                                    |
     |                                    |
     -------------------------------------+
```

As illustrated above, all egress traffic will be routed to Envoy (via iptables) and Envoy will take care of the rest. All ingress traffic, however, will still be directly handled by the service.

### Envoy configuration
#### Egress
* Cluster `proxy_cluster` set as a dynamic forward proxy
* Listener on `127.0.0.1:9001` as egress
* Egress is configured to use the `proxy_cluster` as its upstream cluster

### Verification

```
kubectl apply -f manifests/

# Optional
kubectl apply -f prom/2-prom-crds.yaml
kubectl apply -f prom/2-prom-instance.yaml
```

We then exec into one of the services and run `curl` a few times:

```
kubectl exec -it service-a -c service -- sh
curl service-b:8080 -v
```

You can either check the Envoy stats page (exposed at 8001):

```
# Access Envoy admin via http://localhost:9999
kubectl port-forward service-a 9999:8001
```

Or deploy the optional Prometheus manifest (exposed as a NodePort on 30900) to see the stats scraped from the Envoy sidecars:

```
# If access via NodePort is not possible:
kubectl port-forward prometheus-prometheus-0 9991:9090

increase(envoy_http_downstream_rq_xx{envoy_http_conn_manager_prefix="egress_http"}[1m])
increase(envoy_cluster_upstream_rq_xx{envoy_cluster_name="proxy_cluster"}[1m])
increase(envoy_dns_cache_proxy_dns_cache_dns_query_attempt[1m])
increase(envoy_dns_cache_proxy_dns_cache_dns_query_success[1m])
increase(envoy_dns_cache_proxy_dns_cache_host_added[1m])
```

### Cleaning up

```
kubectl delete -f manifests/
kubectl delete -f prom/
kubectl delete crd prometheusrules.monitoring.coreos.com
kubectl delete crd podmonitors.monitoring.coreos.com
kubectl delete crd alertmanagers.monitoring.coreos.com
kubectl delete crd thanosrulers.monitoring.coreos.com
```

### References

* https://www.envoyproxy.io/docs/envoy/latest/operations/stats_overview
* https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md
