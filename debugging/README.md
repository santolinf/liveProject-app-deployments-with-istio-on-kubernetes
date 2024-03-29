# Debugging the Mesh 

## Using the istioctl Diagnostic Tool

The `istioctl` command-line tool can be used to analyse and
diagnose issues with the mesh configuration and to
manage the overall health of the Istio service mesh.

### Analyse the mesh configuration
The `istioctl analyze` tool, inspects all the Istio 
resources in the cluster in aggregate and flags misconfigurations 
that can lead to undesired behavior.

```shell
istioctl analyze -n online-boutique

✔ No validation issues found when analyzing namespace: online-boutique.

```
The analyser (above) found no issues with the current setup of the cluster.

Now, configure an incorrect Istio resource and see what the analyser tool reports.

Below we configure the *wrong* gateway port.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: frontend-gateway
  namespace: online-boutique
spec:
  selector:
    # use Istio default gateway implementation
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: online-boutique-tls-credential
      hosts:
        - "marketplace.boutiquestore.com"
    - port:
        number: 18080
        name: http
        protocol: HTTP
      hosts:
        - "*"
---
```

```shell
kubectl apply -f debugging/wrong-gateway-port.yaml
```
Run the analyser tool again:
```shell
istioctl analyze -n online-boutique
Error [IST0101] (Gateway frontend-gateway.online-boutique) Referenced credentialName not found: "online-boutique-tls-credential"
Warning [IST0104] (Gateway frontend-gateway.online-boutique) The gateway refers to a port that is not exposed on the workload (pod selector istio=ingressgateway; port 18080)
Error: Analyzers found issues when analyzing namespace: online-boutique.
See https://istio.io/v1.10/docs/reference/config/analysis for more information about causes and resolutions.
```

Please refer to the [Configuration Analysis Messages](https://istio.io/latest/docs/reference/config/analysis/) for a list of the distinct possible error or 
warning messages produced by this analysis.

> You can also use this tool for Istio upgrades by inspecting your 
> existing Istio configuration. To do so, run the analyze command with 
> the newer version of istioctl that you’re targeting to upgrade and surface 
> possible configuration deprecations and behavioral changes.

### Reporting analysis as resource status
There is a configuration option in Istio to turn
on reporting of analysis output and misconfiguration issues
as status fields on various Istio resources.

The same validation that was performed by the `istioctl analyze` command
will now be performed by the Istio control plane.

By default, this status reporting feature is *disabled*.

To enable it:
> On Windows run the following on a Windows command prompt.
```shell
istioctl install --set profile=demo --set meshConfig.accessLogFile=/dev/stdout --set meshConfig.accessLogEncoding=JSON --set values.global.istiod.enableAnalysis=true -y
✔ Istio core installed
✔ Istiod installed
✔ Ingress gateways installed
✔ Egress gateways installed
✔ Installation complete
```

Introduce another misconfiguration in the cluster, such as deleting the frontend-gateway:
```shell
kubectl delete gateway frontend-gateway
gateway.networking.istio.io "frontend-gateway" deleted
```

Look at the status of the virtual service resource `frontend-virtual-service`.
```shell
kubectl get virtualservice frontend-virtual-service -o jsonpath='{.status}'
```
```json
{
  "validationMessages": [
    {
      "documentation_url": "https://istio.io/v1.10/docs/reference/config/analysis/ist0101/?ref=status-controller",
      "level": 3,
      "type": {
        "code": "IST0101"
      }
    },
    {
      "documentation_url": "https://istio.io/v1.10/docs/reference/config/analysis/ist0132/?ref=status-controller",
      "level": 8,
      "type": {
        "code": "IST0132"
      }
    }
  ]
}
```

## Using Envoy Configuration

The Envoy administration interface exposes the Envoy configuration and other capabilities
to modify aspects of the proxy. Such configuration is generated by the Istio control plane.

The Envoy configuration can be accessed for every Proxy on port 15000.
```shell
istioctl dashboard envoy deploy/frontend -n online-boutique
http://localhost:15000
```

It is useful to inspect Envoy configuration in situations where your workload injected with sidecar proxies is not 
receiving traffic or your Istio configuration is not affecting the traffic as expected.

The following is taken from [Istio in Action](https://www.manning.com/books/istio-in-action?query=istio).

Envoy can use a set of APIs to do inline configuration updates without any downtime
or restarts.

Collectively, the following APIs are referred to as the __*xDS*__ services.
Envoy uses the following APIs for dynamic configuration:
* Listener discovery service (__LDS__)—An API that allows Envoy to query what listeners
should be exposed on this proxy.
* Route discovery service (__RDS__)—Part of the configuration for listeners that specifies
which routes to use. This is a subset of LDS for when static and dynamic configuration
should be used.
* Cluster discovery service (__CDS__)—An API that allows Envoy to discover what clusters
and respective configuration for each cluster this proxy should have.
* Endpoint discovery service (__EDS__)—Part of the configuration for clusters that specifies
which endpoints to use for a specific cluster. This is a subset of CDS.
* Secret discovery service (__SDS__)—An API used to distribute certificates. It provides a secure way to transport secrets 
without relying on files on disks or external secret-management systems. Istio only supports secret distribution via the 
secure SDS mechanism and provides individual listeners and clusters with the SDS configuration required to decrypt or 
encrypt when receiving or sending traffic.
* Aggregate discovery service (__ADS__)—A serialized stream of all the changes to the
rest of the APIs. You can use this single API to get all of the changes in order.

Check that the data plane is synchronised with the latest configuration, use the `istioctl proxy-status` command.
```shell
istioctl proxy-status
NAME                                                       CDS        LDS        EDS        RDS          ISTIOD                      VERSION
adservice-85598d856b-gv7vz.online-boutique                 SYNCED     SYNCED     SYNCED     SYNCED       istiod-5cbf9df9fd-mg8hh     1.10.3
cartservice-c77f6b866-f4q7s.online-boutique                SYNCED     SYNCED     SYNCED     SYNCED       istiod-5cbf9df9fd-mg8hh     1.10.3
checkoutservice-654c47f4b6-trds2.online-boutique           SYNCED     SYNCED     SYNCED     SYNCED       istiod-5cbf9df9fd-mg8hh     1.10.3
currencyservice-59bc889674-nbzhc.online-boutique           SYNCED     SYNCED     SYNCED     SYNCED       istiod-5cbf9df9fd-mg8hh     1.10.3
emailservice-5b9fff7cb8-qx9sl.online-boutique              SYNCED     SYNCED     SYNCED     SYNCED       istiod-5cbf9df9fd-mg8hh     1.10.3
frontend-75b5c6f8c4-2gmh9.online-boutique                  SYNCED     SYNCED     SYNCED     SYNCED       istiod-5cbf9df9fd-mg8hh     1.10.3
istio-egressgateway-5547fcc8fc-8hlsl.istio-system          SYNCED     SYNCED     SYNCED     NOT SENT     istiod-5cbf9df9fd-mg8hh     1.10.3
istio-ingressgateway-5964bcd6bb-pbf9m.istio-system         SYNCED     SYNCED     SYNCED     SYNCED       istiod-5cbf9df9fd-mg8hh     1.10.3
loadgenerator-6958f5bc8b-w5ftc.online-boutique             SYNCED     SYNCED     SYNCED     SYNCED       istiod-5cbf9df9fd-mg8hh     1.10.3
paymentservice-68dd9755bb-6qtvx.online-boutique            SYNCED     SYNCED     SYNCED     SYNCED       istiod-5cbf9df9fd-mg8hh     1.10.3
productcatalogservice-557ff44b96-qdc28.online-boutique     SYNCED     SYNCED     SYNCED     SYNCED       istiod-5cbf9df9fd-mg8hh     1.10.3
recommendationservice-64dc9dfbc8-6wgsw.online-boutique     SYNCED     SYNCED     SYNCED     SYNCED       istiod-5cbf9df9fd-mg8hh     1.10.3
redis-cart-5b569cd47-s8q6v.online-boutique                 SYNCED     SYNCED     SYNCED     SYNCED       istiod-5cbf9df9fd-mg8hh     1.10.3
shippingservice-5488d5b6cb-dz7bm.online-boutique           SYNCED     SYNCED     SYNCED     SYNCED       istiod-5cbf9df9fd-mg8hh     1.10.3
```

#### Examples
Look at the Envoy listener configuration for the Istio ingress gateway pod.

```shell
istioctl proxy-config listeners deploy/istio-ingressgateway -n istio-system
ADDRESS PORT  MATCH                              DESTINATION
0.0.0.0 8080  ALL                                Route: http.80
0.0.0.0 8443  SNI: marketplace.boutiquestore.com Route: https.443.https.frontend-gateway.online-boutique
0.0.0.0 15021 ALL                                Inline Route: /healthz/ready*
0.0.0.0 15090 ALL                                Inline Route: /stats/prometheus*
```

The first two listeners correspond to the frontend gateway resource defined in the `online-boutique` namespace
to have external requests target the boutique store services, at ports `80` and `443`. The Istio control
plane does the translation to the target ports of `8080` and `8443`.

Both listeners have filters, an HTTP filter that points to the route name `http.80` and an HTTPS filter
that points to the route name `https.443.https.frontend-gateway.online-boutique`.

The other two listeners are for health monitoring and for Prometheus to collect metrics from the
gateway.

You can get more information about a specific listener and reveal the filter chain configuration.
```shell
istioctl proxy-config listeners deploy/istio-ingressgateway -n istio-system --address "0.0.0.0" --port 8080 -o json
```
The output is rather verbose, but below I show an excerpt.
```
   ...
        "filterChains": [
            {
                "filters": [
                    {
                        "name": "envoy.filters.network.http_connection_manager",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                            "statPrefix": "outbound_0.0.0.0_8080",
                            "rds": {
                                "configSource": {
                                    "ads": {},
                                    "initialFetchTimeout": "0s",
                                    "resourceApiVersion": "V3"
                                },
                                "routeConfigName": "http.80"
                            },
                            "httpFilters": [
                                {
                                    "name": "istio.metadata_exchange",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                                        "typeUrl": "type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm",
                                        "value": {
                                            "config": {
                                                "configuration": {
                                                    "@type": "type.googleapis.com/google.protobuf.StringValue",
                                                    "value": "{}\n"
                                                },
                                                "vm_config": {
                                                    "code": {
                                                        "local": {
                                                            "inline_string": "envoy.wasm.metadata_exchange"
                                                        }
                                                    },
                                                    "runtime": "envoy.wasm.runtime.null"
                                                }
                                            }
                                        }
                                    }
                                },
                                {
                                    "name": "envoy.filters.http.jwt_authn",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication",
                                        "providers": {
                                            "origins-0": {
                                                "issuer": "testing@secure.istio.io",
                                                "localJwks": {
                                                    "inlineString": "{ \"keys\":[ {\"e\":\"AQAB\",\"kid\":\"DHFbpoIUqrY8t2zpA2qXfCmr5VO5ZEr4RzHU_-envvQ\",\"kty\":\"RSA\",\"n\":\"xAE7eB6qugXyCAG3yhh7pkDkT65pHymX-P7KfIupjf59vsdo91bSP9C8H07pSAGQO1MV_xFj9VswgsCg4R6otmg5PV2He95lZdHtOcU5DXIg_pbhLdKXbi66GlVeK6ABZOUW3WYtnNHD-91gVuoeJT_DwtGGcp4ignkgXfkiEm4sw-4sfb4qdt5oLbyVpmW6x9cfa7vs2WTfURiCrBoUqgBo_-4WTiULmmHSGZHOjzwa8WtrtOQGsAFjIbno85jp6MnGGGZPYZbDAa_b3y5u-YpW7ypZrvD8BgtKVjgtQgZhLAGezMt0ua3DRrWnKqTZ0BJ_EyxOGuHJrLsn00fnMQ\"}]}"
                                                },
                                                "payloadInMetadata": "testing@secure.istio.io"
                                            }
                                        },
                                        "rules": [
                                            {
                                                "match": {
                                                    "prefix": "/"
                                                },
                                                "requires": {
                                                    "requiresAny": {
                                                        "requirements": [
                                                            {
                                                                "providerName": "origins-0"
                                                            },
                                                            {
                                                                "allowMissing": {}
                                                            }
                                                        ]
                                                    }
                                                }
                                            }
                                        ]
                                    }
                                },
                                {
                                    "name": "istio_authn",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/istio.envoy.config.filter.http.authn.v2alpha1.FilterConfig",
                                        "policy": {
                                            "origins": [
                                                {
                                                    "jwt": {
                                                        "issuer": "testing@secure.istio.io"
                                                    }
                                                }
                                            ],
                                            "originIsOptional": true,
                                            "principalBinding": "USE_ORIGIN"
                                        },
                                        "skipValidateTrustDomain": true
                                    }
                                },
                                {
                                    "name": "envoy.filters.http.rbac",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.extensions.filters.http.rbac.v3.RBAC",
                                        "rules": {
                                            "action": "DENY",
                                            "policies": {
                                                "ns[istio-system]-policy[online-boutique]-rule[0]": {
                                                    "permissions": [
                                                        {
                                                            "andRules": {
                                                                "rules": [
                                                                    {
                                                                        "orRules": {
                                                                            "rules": [
                                                                                {
                                                                                    "urlPath": {
                                                                                        "path": {
                                                                                            "exact": "/cart"
                                                                                        }
                                                                                    }
                                                                                }
                                                                            ]
                                                                        }
                                                                    },
                                                                    {
                                                                        "orRules": {
                                                                            "rules": [
                                                                                {
                                                                                    "destinationPort": 8443
                                                                                }
                                                                            ]
                                                                        }
                                                                    }
                                                                ]
                                                            }
                                                        }
                                                    ],
                                                    "principals": [
                                                        {
                                                            "andIds": {
                                                                "ids": [
                                                                    {
                                                                        "notId": {
                                                                            "orIds": {
                                                                                "ids": [
                                                                                    {
                                                                                        "metadata": {
                                                                                            "filter": "istio_authn",
                                                                                            "path": [
                                                                                                {
                                                                                                    "key": "request.auth.audiences"
                                                                                                }
                                                                                            ],
                                                                                            "value": {
                                                                                                "stringMatch": {
                                                                                                    "exact": "boutiquestore.com"
                                                                                                }
                                                                                            }
                                                                                        }
                                                                                    }
                                                                                ]
                                                                            }
                                                                        }
                                                                    }
                                                                ]
                                                            }
                                                        }
                                                    ]
                                                }
                                            }
                                        },
                                        "shadowRulesStatPrefix": "istio_dry_run_allow_"
                                    }
                                },
                                {
                                    "name": "envoy.filters.http.cors",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.extensions.filters.http.cors.v3.Cors"
                                    }
                                },
                                {
                                    "name": "envoy.filters.http.fault",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.extensions.filters.http.fault.v3.HTTPFault"
                                    }
                                },
                                {
                                    "name": "istio.stats",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                                        "typeUrl": "type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm",
                                        "value": {
                                            "config": {
                                                "configuration": {
                                                    "@type": "type.googleapis.com/google.protobuf.StringValue",
                                                    "value": "{\n  \"debug\": \"false\",\n  \"stat_prefix\": \"istio\",\n  \"disable_host_header_fallback\": true\n}\n"
                                                },
                                                "root_id": "stats_outbound",
                                                "vm_config": {
                                                    "code": {
                                                        "local": {
                                                            "inline_string": "envoy.wasm.stats"
                                                        }
                                                    },
                                                    "runtime": "envoy.wasm.runtime.null",
                                                    "vm_id": "stats_outbound"
                                                }
                                            }
                                        }
                                    }
                                },
                                {
                                    "name": "envoy.filters.http.router",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
                                    }
                                }
                            ],
   ...
```

Next, look up the route `http.80` using the following command:
```shell
istioctl proxy-config routes deploy/istio-ingressgateway -n istio-system --name http.80 -o json
```
```json
[
    {
        "name": "http.80",
        "virtualHosts": [
            {
                "name": "*:80",
                "domains": [
                    "*"
                ],
                "routes": [
                    {
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "outbound|80||frontend.online-boutique.svc.cluster.local",
                            "timeout": "0s",
                            "retryPolicy": {
                                "retryOn": "connect-failure,refused-stream,unavailable,cancelled,retriable-status-codes",
                                "numRetries": 2,
                                "retryHostPredicate": [
                                    {
                                        "name": "envoy.retry_host_predicates.previous_hosts"
                                    }
                                ],
                                "hostSelectionRetryMaxAttempts": "5",
                                "retriableStatusCodes": [
                                    503
                                ]
                            },
                            "maxGrpcTimeout": "0s"
                        },
                        "metadata": {
                            "filterMetadata": {
                                "istio": {
                                    "config": "/apis/networking.istio.io/v1alpha3/namespaces/online-boutique/virtual-service/frontend-virtual-service"
                                }
                            }
                        },
                        "decorator": {
                            "operation": "frontend.online-boutique.svc.cluster.local:80/*"
                        }
                    }
                ],
                "includeRequestAttemptCount": true
            }
        ],
        "validateClusters": false
    }
]
```

The virtual host includes a single cluster. All traffic is sent to this cluster, because of the single route
matching all requests.

    "cluster": "outbound|80||frontend.online-boutique.svc.cluster.local"

Next, look at the cluster configuration:
```shell
istioctl proxy-config clusters deploy/istio-ingressgateway -n istio-system --fqdn 'frontend.online-boutique.svc.cluster.local' -o json 
```

```json
[
    {
        "transportSocketMatches": [
            {
                "name": "tlsMode-istio",
                "match": {
                    "tlsMode": "istio"
                },
                "transportSocket": {
                    "name": "envoy.transport_sockets.tls",
                    "typedConfig": {
                        "@type": "type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext",
                        "commonTlsContext": {
                            "tlsCertificateSdsSecretConfigs": [
                                {
                                    "name": "default",
                                    "sdsConfig": {
                                        "apiConfigSource": {
                                            "apiType": "GRPC",
                                            "transportApiVersion": "V3",
                                            "grpcServices": [
                                                {
                                                    "envoyGrpc": {
                                                        "clusterName": "sds-grpc"
                                                    }
                                                }
                                            ],
                                            "setNodeOnFirstMessageOnly": true
                                        },
                                        "initialFetchTimeout": "0s",
                                        "resourceApiVersion": "V3"
                                    }
                                }
                            ],
                            "combinedValidationContext": {
                                "defaultValidationContext": {},
                                "validationContextSdsSecretConfig": {
                                    "name": "ROOTCA",
                                    "sdsConfig": {
                                        "apiConfigSource": {
                                            "apiType": "GRPC",
                                            "transportApiVersion": "V3",
                                            "grpcServices": [
                                                {
                                                    "envoyGrpc": {
                                                        "clusterName": "sds-grpc"
                                                    }
                                                }
                                            ],
                                            "setNodeOnFirstMessageOnly": true
                                        },
                                        "initialFetchTimeout": "0s",
                                        "resourceApiVersion": "V3"
                                    }
                                }
                            },
                            "alpnProtocols": [
                                "istio-peer-exchange",
                                "istio"
                            ]
                        },
                        "sni": "outbound_.80_._.frontend.online-boutique.svc.cluster.local"
                    }
                }
            },
            {
                "name": "tlsMode-disabled",
                "match": {},
                "transportSocket": {
                    "name": "envoy.transport_sockets.raw_buffer"
                }
            }
        ],
        "name": "outbound|80||frontend.online-boutique.svc.cluster.local",
        "type": "EDS",
        "edsClusterConfig": {
            "edsConfig": {
                "ads": {},
                "initialFetchTimeout": "0s",
                "resourceApiVersion": "V3"
            },
            "serviceName": "outbound|80||frontend.online-boutique.svc.cluster.local"
        },
        "connectTimeout": "10s",
        "circuitBreakers": {
            "thresholds": [
                {
                    "maxConnections": 4294967295,
                    "maxPendingRequests": 4294967295,
                    "maxRequests": 4294967295,
                    "maxRetries": 4294967295,
                    "trackRemaining": true
                }
            ]
        },
        "metadata": {
            "filterMetadata": {
                "istio": {
                    "default_original_port": 80,
                    "services": [
                        {
                            "host": "frontend.online-boutique.svc.cluster.local",
                            "name": "frontend",
                            "namespace": "online-boutique"
                        }
                    ]
                }
            }
        },
        "filters": [
            {
                "name": "istio.metadata_exchange",
                "typedConfig": {
                    "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                    "typeUrl": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
                    "value": {
                        "protocol": "istio-peer-exchange"
                    }
                }
            }
        ]
    }
]
```
The endpoints of this cluster will be fetched via EDS.
The corresponding EDS service name to get the endpoints associated with this cluster is 
`outbound|80||frontend.online-boutique.svc.cluster.local`.

Next, look at the endpoints configuration:
```shell
istioctl proxy-config endpoints deploy/istio-ingressgateway -n istio-system --cluster 'outbound|80||frontend.online-boutique.svc.cluster.local' -o json
```
```json
[
    {
        "name": "outbound|80||frontend.online-boutique.svc.cluster.local",
        "addedViaApi": true,
        "hostStatuses": [
            {
                "address": {
                    "socketAddress": {
                        "address": "10.1.37.122",
                        "portValue": 8080
                    }
                },
                "stats": [
                    {
                        "name": "cx_connect_fail"
                    },
                    {
                        "name": "cx_total"
                    },
                    {
                        "name": "rq_error"
                    },
                    {
                        "name": "rq_success"
                    },
                    {
                        "name": "rq_timeout"
                    },
                    {
                        "name": "rq_total"
                    },
                    {
                        "type": "GAUGE",
                        "name": "cx_active"
                    },
                    {
                        "type": "GAUGE",
                        "name": "rq_active"
                    }
                ],
                "healthStatus": {
                    "edsHealthStatus": "HEALTHY"
                },
                "weight": 1,
                "locality": {}
            }
        ],
        "circuitBreakers": {
            "thresholds": [
                {
                    "maxConnections": 4294967295,
                    "maxPendingRequests": 4294967295,
                    "maxRequests": 4294967295,
                    "maxRetries": 4294967295
                },
                {
                    "priority": "HIGH",
                    "maxConnections": 1024,
                    "maxPendingRequests": 1024,
                    "maxRequests": 1024,
                    "maxRetries": 3
                }
            ]
        },
        "observabilityName": "outbound|80||frontend.online-boutique.svc.cluster.local"
    }
]
```

The hosts contain the exact address and port of the frontend pod where the traffic will be routed from the 
Istio ingress gateway.
You can verify that the IP address above matches the frontend pod’s IP address as shown:
```shell
kubectl get pod -n online-boutique -l app=frontend -o wide
NAME                        READY   STATUS    RESTARTS      AGE   IP            NODE         NOMINATED NODE   READINESS GATES
frontend-75b5c6f8c4-2gmh9   2/2     Running   6 (41m ago)   35h   10.1.37.122   kubeserver   <none>           <none>
```

## <a id="milestone6"></a> Deliverable for this Milestone

Scale up the frontend deployment in the online-boutique namespace to two replicas and submit the output 
of the command `istioctl proxy-config endpoint` for the Istio ingress gateway pod corresponding to the 
frontend cluster, which should show two hosts related to the old and the new replica.

Confirm that we have only one replica to start with.
```shell
kubectl get pods -n online-boutique -l app=frontend
NAME                        READY   STATUS    RESTARTS      AGE
frontend-75b5c6f8c4-2gmh9   2/2     Running   6 (79m ago)   35h
```

Now, scale up the replicas.
```shell
kubectl scale --replicas=2 deploy/frontend -n online-boutique
deployment.apps/frontend scaled
```

Verify that there are now two replicas running.
```shell
kubectl get pods -n online-boutique -l app=frontend
NAME                        READY   STATUS    RESTARTS      AGE
frontend-75b5c6f8c4-2gmh9   2/2     Running   6 (83m ago)   35h
frontend-75b5c6f8c4-tszrx   2/2     Running   0             25s
```

The Endpoints for the frontend cluster should now report two hosts, one being the old replica and the
other being the new scaled up replica.
```shell
istioctl proxy-config endpoints deploy/istio-ingressgateway -n istio-system --cluster 'outbound|80||frontend.online-boutique.svc.cluster.local'
ENDPOINT             STATUS      OUTLIER CHECK     CLUSTER
10.1.37.108:8080     HEALTHY     OK                outbound|80||frontend.online-boutique.svc.cluster.local
10.1.37.122:8080     HEALTHY     OK                outbound|80||frontend.online-boutique.svc.cluster.local
```

```shell
kubectl get pod -n online-boutique -l app=frontend -o wide
NAME                        READY   STATUS    RESTARTS      AGE    IP            NODE         NOMINATED NODE   READINESS GATES
frontend-75b5c6f8c4-2gmh9   2/2     Running   6 (88m ago)   35h    10.1.37.122   kubeserver   <none>           <none>
frontend-75b5c6f8c4-tszrx   2/2     Running   0             6m3s   10.1.37.108   kubeserver   <none>           <none>
```
