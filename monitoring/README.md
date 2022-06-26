
```shell
kubectl -n monitoring port-forward $(kubectl -n monitoring get po -l app=prometheus --no-headers -o jsonpath='{.items[0].metadata.name}') 9090:9090
```

```shell
curl -sS --data-urlencode query="sum by (destination_app, destination_service, response_code, reporter) (increase(istio_requests_total{source_app=\"istio-ingressgateway\", destination_app!=\"unknown\", reporter=\"source\"}[5m]))" "http://localhost:9090/api/v1/query" | jq .
```
