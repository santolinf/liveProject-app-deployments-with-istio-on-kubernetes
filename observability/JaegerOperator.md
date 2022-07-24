# microk8s Jaeger add-on

These are notes about the __microk8s__ add-on *Jaeger* simplest implementation
of the [Jaeger Operator](https://www.jaegertracing.io/docs/1.24/operator/).

These are some steps to configure your apps when Istio isn't involved.

## Jaeger Operator

I have used Version 1.24.

According to Jaeger Operator documentation

> It is possible to have the Jaeger Operator running in a given namespace (like, observability) and manage Jaeger resources
> in another (like, myproject). For that, use a RoleBinding like the following for each namespace the operator should watch for resources

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: jaeger-operator-in-online-boutique
  namespace: online-boutique
subjects:
- kind: ServiceAccount
  name: jaeger-operator
  namespace: default
roleRef:
  kind: Role
  name: jaeger-operator
  apiGroup: rbac.authorization.k8s.io
```

Apply the RoleBinding resources to both the `istio-system` and `online-boutique` namespaces.
```shell
kubectl apply -f monitoring/jaeger-op-in-istio-system.yaml
```
```shell
kubectl apply -f monitoring/jaeger-op-in-online-boutique.yaml
```

### Patching sidecars
```shell
kubectl get deployment frontend -o jsonpath='{.metadata.annotations}'
```
```shell
kubectl annotate deployment frontend "sidecar.jaegertracing.io/inject=true"
deployment.apps/frontend annotated
```

https://www.jaegertracing.io/docs/1.36/operator/

https://discuss.kubernetes.io/t/jaeger-auto-injecting-into-pods/8138
