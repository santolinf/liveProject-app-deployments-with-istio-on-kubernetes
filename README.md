# liveProject-app-deployments-with-istio-on-kubernetes

## Disable Istio access logging
> On Windows run the following on a Windows command prompt.
```shell
istioctl install --set profile=demo --set meshConfig.accessLogFile= --set meshConfig.accessLogEncoding=TEXT -y
```
