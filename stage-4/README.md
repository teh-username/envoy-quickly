## "Deploying" on an actual application

Let's now circle back to the original motivation for this idea, setting up metrics quickly on an already existing application.

For this purpose, we'll be using [Bouyant's](https://buoyant.io/) sample app, [emojivoto](https://github.com/BuoyantIO/emojivoto) (which is also used by [Linkerd](https://linkerd.io/) as its demo app).

There's also [Istio's](https://istio.io/) [bookinfo app](https://github.com/istio/istio/blob/master/samples/bookinfo/platform/kube/bookinfo.yaml) but emojivoto comes with a traffic generator, so we'll go with emojivoto. Exercise for the reader perhaps?

Once fully deployed, our application should look like:

```
                                                               +------------------+
                                                               |+-----+    +-----+|
                                                           --> ||proxy|<-->|emoji||
+---------------------+        +----------------+    -----/    |+-----+    +-----+|
|+--------+    +-----+|        |+-----+    +---+|---/          +------------------+
||vote-bot|<-->|proxy||------> ||proxy|<-->|web||
|+--------+    +-----+|        |+-----+    +---+|---\          +-------------------+
+---------------------+        +----------------+    -----\    |+-----+    +------+|
                                                           --> ||proxy|<-->|voting||
                                                               |+-----+    +------+|
                                                               +-------------------+
```

No control plane!

### Envoy Configuration

#### web-svc config (web-envoy-config)
##### Ingress
* Cluster `local_service` pointed at `127.0.0.1:8080`
* Listener on `0.0.0.0:9211` as ingress
* Ingress is configured to use `local_service` as its upstream cluster

##### Egress
* Cluster `proxy_cluster` set as a dynamic forward proxy, uses HTTP/2 (GRPC)
* Listener on `127.0.0.1:9001` as egress
* Egress is configured to use the `proxy_cluster` as its upstream cluster

#### vote-bot, voting-svc and emoji-svc config (envoy-config)
##### Ingress
* Cluster `local_service` pointed at `127.0.0.1:8080`, uses HTTP/2 (GRPC)
* Listener on `0.0.0.0:9211` as ingress
* Ingress is configured to use `local_service` as its upstream cluster

_Note: vote-bot has no ingress so we can leave the HTTP/2 support as is without any impact_

##### Egress
* Cluster `proxy_cluster` set as a dynamic forward proxy
* Listener on `127.0.0.1:9001` as egress
* Egress is configured to use the `proxy_cluster` as its upstream cluster

_Note: voting-svc and emoji-svc doesn't talk to each other so we don't have to set HTTP/2 at egress_

### Verification

We can retrieve emojivoto's entire manifest by:

```
kubectl kustomize github.com/BuoyantIO/emojivoto/kustomize/deployment > emoji-voto.yaml
```

`manifests/` already has each service (injected with Envoy) on its own file for easier reading.

Deploy the app:

```
kubectl apply -f manifests/
kubectl apply -f prom/2-prom-crds.yaml
kubectl apply -f prom/2-prom-instance.yaml
```

Access to Prometheus is exposed via NodePort on 30900. Try querying for the following:

```
increase(envoy_cluster_external_upstream_rq_xx{envoy_cluster_name="proxy_cluster"}[1m])

  sum(rate(envoy_http_downstream_rq_time_bucket{envoy_http_conn_manager_prefix="egress_http", le="1", service="web-svc"}[2m])) by (job)
/
  sum(rate(envoy_http_downstream_rq_time_count{envoy_http_conn_manager_prefix="egress_http", service="web-svc"}[2m])) by (job)

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

References:

* https://github.com/BuoyantIO/emojivoto
* https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
