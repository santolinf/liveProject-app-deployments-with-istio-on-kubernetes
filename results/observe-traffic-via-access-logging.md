## Configuring access logging

Turn on Envoy access-logging globally mesh-wide.

```shell
istioctl install --set profile=demo --set meshConfig.accessLogFile="/dev/stdout" --set meshConfig.accessLogEncoding=JSON -y
```

The above command turns on access logging and sends the logs to standard out (`/dev/stdout`),
and sets the access logs encoding to JSON.

### Generate logs for a request to a non-existent path

```shell
curl http://$INGRESS/wrongpath
```
```shell
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    19  100    19    0     0    268      0 --:--:-- --:--:-- --:--:--   275404 page not found

```

The response of the above cUrl command is `404 page not found`.

### Examine the ingress gateway access logs

```shell
kubectl -n istio-system logs -l app=istio-ingressgateway
```
```json
{
  "downstream_local_address": "10.1.37.69:8080",
  "path": "/wrongpath",
  "method": "GET",
  "upstream_transport_failure_reason": null,
  "response_flags": "-",
  "user_agent": "curl/7.77.0",
  "connection_termination_details": null,
  "request_id": "5f47fdad-50a2-9352-b972-00ff681d5b2e",
  "upstream_cluster": "outbound|80||frontend.online-boutique.svc.cluster.local",
  "authority": "192.168.1.41",
  "requested_server_name": null,
  "upstream_local_address": "10.1.37.69:44134",
  "response_code": 404,
  "x_forwarded_for": "192.168.1.211",
  "downstream_remote_address": "192.168.1.211:46330",
  "bytes_sent": 19,
  "protocol": "HTTP/1.1",
  "upstream_host": "10.1.37.107:8080",
  "bytes_received": 0,
  "upstream_service_time": "40",
  "start_time": "2022-06-03T17:32:57.106Z",
  "route_name": null,
  "duration": 41,
  "response_code_details": "via_upstream"
}

```

The request is proxied to the `frontend` service as we can see from the `upstream_host` field value `"upstream_host": "10.1.37.107:8080"`.
To add to this, the `upstream_cluster` field value is `"upstream_cluster": "outbound|80||frontend.online-boutique.svc.cluster.local"`
which indicates that the Istio ingress gateway has routed the request to
the target `frontend` service.

```shell
$ kubectl describe svc frontend
Name:              frontend
Namespace:         online-boutique
Labels:            <none>
Annotations:       <none>
Selector:          app=frontend
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.152.183.105
IPs:               10.152.183.105
Port:              http  80/TCP
TargetPort:        8080/TCP
Endpoints:         10.1.37.107:8080
Session Affinity:  None
Events:            <none>

```
