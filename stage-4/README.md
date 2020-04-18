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
#### Ingress
* Cluster
  * `grpc_local_service` pointed at `127.0.0.1:8080`, uses HTTP/2 (GRPC)
  * `rest_local_service` pointed at `127.0.0.1:8080`, defaults to HTTP/1.x

* Listener
  * `0.0.0.0:9211` as ingress with the following routes:
    * `grpc_route` (if "application/grpc") routes to `grpc_local_service`
    * `rest_route` (default route) routes to `rest_local_service`

#### Egress
* Cluster
  * `grpc_proxy_cluster` set as a dynamic forward proxy, uses HTTP/2 (GRPC)
  * `rest_proxy_cluster` set as a dynamic forward proxy, defaults to HTTP/1.x

* Listener
  * `127.0.0.1:9001` as egress with the following routes:
    * `grpc_route` (if "application/grpc") routes to `grpc_proxy_cluster`
    * `rest_route` (default route) routes to `rest_proxy_cluster`

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
increase(envoy_cluster_external_upstream_rq_xx{envoy_cluster_name="grpc_cluster"}[1m])
increase(envoy_cluster_external_upstream_rq_xx{envoy_cluster_name="rest_cluster"}[1m])

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

* https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md
* https://github.com/BuoyantIO/emojivoto
* https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
