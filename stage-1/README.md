## Envoy as a forward proxy

For our first stage, we'll set up the following using Docker Compose:

```
                 +---------+
                 |service-b|
                 +---------+
                      ^
                      |
                      v
+---------+        +-----+        +---------+
|service-a|<------>|proxy|<------>|service-c|
+---------+        +-----+        +---------+
```

The goal is for the services to be able to talk to each other by routing each request via the proxy, i.e.

`curl proxy:9000 -H "Host: <service>:<port>"`

without explicitly adding each service as a cluster in our Envoy configuration.

We can achieve this by configuring our upstream cluster as follows:

```
clusters:
  - name: proxy_cluster
    connect_timeout: 1s
    lb_policy: CLUSTER_PROVIDED
    cluster_type:
      name: envoy.clusters.dynamic_forward_proxy
      typed_config:
        "@type": type.googleapis.com/envoy.config.cluster.dynamic_forward_proxy.v2alpha.ClusterConfig
        dns_cache_config:
          name: proxy_dns_cache
          dns_lookup_family: V4_ONLY
```

Envoy will simply do DNS resolution (asynchronously) if the host is not yet in the cache.

### Verification

1) Run `docker-compose up`
2) Docker exec to each of the services and then run the curl command above (substitute the correct host and port [web server is on port 8080 for all services])

Each service should be reachable via the proxy container. You may also check the stats of the Envoy proxy and config dump (exposed on port 8001) for further sanity checks.

### References

* https://github.com/envoyproxy/envoy/issues/8768
* https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/dynamic_forward_proxy_filter
