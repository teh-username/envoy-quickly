## Proxying ingress and egress traffic

In this stage, Envoy will now be handling both ingress and egress traffic of the services:

```
+-----------------------+        +-----------------------+
|+---------+     +-----+|        |+-----+     +---------+|
||service-a|<--->|proxy||<------>||proxy|<--->|service-b||
|+---------+     +-----+|        |+-----+     +---------+|
+-----------------------+        +-----------------------+
```

No matter how many new services we deploy, our Envoy configuration need not change. This effectively removes the control plane out of the picture.

### Envoy Configuration

#### Ingress
* Cluster `local_service` pointed at `127.0.0.1:8080`
* Listener on `0.0.0.0:9211` as ingress
* Ingress is configured to use `local_service` as its upstream cluster

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

We can then generate traffic as follows:

```
kubectl exec -it service-a -c service -- sh
curl service-b:8080 -v

kubectl exec -it service-b -c service -- sh
curl service-a:8080 -v
```

Access to Envoy stats:

```
# Access Envoy admin via http://localhost:9999
kubectl port-forward service-a 9999:8001
```

Access to Prometheus is exposed via NodePort on 30900. If deployed, try querying for the following stats:

```
increase(envoy_http_downstream_rq_xx{envoy_http_conn_manager_prefix=~".*http"}[1m])
increase(envoy_cluster_upstream_rq_xx[1m])
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
