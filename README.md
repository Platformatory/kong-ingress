# Changes to docker-compose 

1. Configured database info to use the postgres DB in the network.
2. Additional options to `KONG_ADMIN_GUI_AUTH`. Added `KONG_ADMIN_GUI_SESSION_CONF`.
3. Changed the port of kong admin API service from 8010 to 8001.
4. Updated `KONG_ADMIN_GUI_URL` and `KONG_ADMIN_API_URI` to point to localhost.

# Activity 1

Mark one of the targets unhealthy.

```
$ curl  -H 'Kong-Admin-Token:password' -X POST http://localhost:8001/upstreams/httpbin-upstream/targets/localhost:80/unhealthy
```

Issue an API call to upstream.

```
$ curl http://localhost:8000/echo
{
  "args": {}, 
  "data": "", 
  "files": {}, 
  "form": {}, 
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin.org", 
    "User-Agent": "curl/7.68.0", 
    "X-Amzn-Trace-Id": "Root=1-621ca28d-3d4a2981179e472957d5f5ba", 
    "X-Forwarded-Host": "localhost", 
    "X-Forwarded-Path": "/echo", 
    "X-Forwarded-Prefix": "/echo"
  }, 
  "json": null, 
  "method": "GET", 
  "origin": "172.29.0.1, 49.37.210.211", 
  "url": "http://localhost/anything"
}
```

Mark other target unhealthy.

```
$ curl  -H 'Kong-Admin-Token:password' -X POST http://localhost:8001/upstreams/httpbin-upstream/targets/httpbin.org:80/unhealthy
```

API doesn't respond.

```
$ curl http://localhost:8000/echo
{"message":"failure to get a peer from the ring-balancer"}
```

Mark original target as healthy.

```
curl  -H 'Kong-Admin-Token:password' -X POST http://localhost:8001/upstreams/httpbin-upstream/targets/localhost:80/healthy
```

# Activity 2

Install kong ingress controller.

```
helm install kong/kong --generate-name --set ingressController.installCRDs=false
export PROXY_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" service kong-1646046879-kong-proxy)
```

Add key auth plugin.

```
kubectl apply -f keyauth-plugin.yml
```

Configure key auth for service.

```
kubectl patch service echo -p '{"metadata":{"annotations":{"konghq.com/plugins":"echo-auth"}}}'
```

Create 2 consumers.

```
kubectl apply -f lakshmi-consumer.yml
kubectl apply -f dasa-consumer.yml
```

Create API key for consumers.

```
kubectl create secret generic lakshmi-apikey  \
  --from-literal=kongCredType=key-auth  \
  --from-literal=key=lakshmi-secret-key

kubectl create secret generic dasa-apikey  \
  --from-literal=kongCredType=key-auth  \
  --from-literal=key=dasa-secret-key
```

Update API key for consumers.

```
kubectl apply -f lakshmi-consumer.yml
kubectl apply -f dasa-consumer.yml
```

Try out API with API key.

```
curl -i -H 'apikey: lakshmi-secret-key' $PROXY_IP/foo
```

Add 2 rate limiting plugins, one for each consumer.

```
kubectl apply -f rate-limit-1.yml
kubectl apply -f rate-limit-5.yml
```

Configure rate limits for both consumers.

```
kubectl apply -f lakshmi-consumer.yml
kubectl apply -f dasa-consumer.yml
```

Test rate limits.
