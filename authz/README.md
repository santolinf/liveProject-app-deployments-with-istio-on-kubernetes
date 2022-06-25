# Adding User Authentication

The Istio ingress gateway can be configured to perform user authentication for the whole mesh, that is all workloads on
all namespaces.
> Note that authentication can also be performed at other scope levels: namespace and workload scopes

Istio's `RequestAuthentication` resource defines the authentication mechanism used for verifying the user's identity.
Here we are implementing JWT validation as the mechanism with which the user identity will be verified.

The Istio ingress gateway will validate the user authentication credentials (i.e., JWT) with every request and route the request to
the target service. The Istio ingress gateway will reject the request instead of routing to the target service 
when an invalid JWT is present. However, when the request has no authentication credentials then the request is allowed
but it will not have any authenticated identity. 

`RequestAuthentication` resource validates the provided authentication credentials, extract the claims of valid tokens,
and store those claims in *filter metadata* (set of key value pairs available in the service proxy).

> The Istio AuthN Filter saves authentication results (i.e., JWT claims) to Envoy dynamic metadata for use 
> in later filters (e.g. authorization)

Authorization in Istio is configured via `AuthorizationPolicy` resources, which uses any available filter metadata.

To enforce strict access control, both of these resources should be used in conjunction.

## Demo JWTs and JWKS endpoints

Before starting, set up your environment with the following Demo tokens. Also note that the Demo JWKS endpoint is as follows:

    https://raw.githubusercontent.com/istio/istio/release-1.9/security/tools/jwt/samples/jwks.json

```shell
export VALID_TOKEN="eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJhdWQiOiJib3V0aXF1ZXN0b3JlLmNvbSIsImV4cCI6MTkxNDM2NTM2NSwiaWF0IjoxNTk5MDA1MzY1LCJpc3MiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyIsInN1YiI6InVzZXIxQHh5ei5jb20ifQ.pZHGJgEP0qsJ1pSBPU6nwpMETf3MqlWx-T0PED0BLb5aTnbXWklIzHV4JHgMk6Td4thI4cC4_mCvrFlgm1uqzO2U_DEdNOReNMGNnDSQ87hWEtOQxn50c83dVq3-w4FUrKBjS1lYhiabKWVfr3BC_Bmwefl1TVhgP_iNuQGsWsj_Tcl4TWP93yS93DazphAl0uIOw3WvmJSfhnaZ7cEIk0PzGmsLq96FLbGatZX1DL8OKhv2Hz-iN0V6KG_5GjjEdD7LVSprw4oedtwbahDVVuxh0due1DImXLhH9oH1_RkntrD98JlKYoNzG1-7apSV0xw5mlE8L2HXDo3TKE33WA"
```

```shell
export INVALID_TOKEN="eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJhdWQiOiJib3V0aXF1ZXN0b3JlLmNvbSIsImV4cCI6MTkxNDM2NTI1MSwiaWF0IjoxNTk5MDA1MjUxLCJpc3MiOiJ3cm9uZy1pc3N1c2VyQHNlY3VyZS5pc3Rpby5pbyIsInN1YiI6InVzZXIxQHh5ei5jb20ifQ.SnQPGlkV66Q61zR8uZAzEhyPfynNmV_MGzvjnkxZhVv-elrKu7Wq50tj4SKGyjTJPR2-YFd_-p3eN4VveCH5NB3LjgyOliMQjnxSTN92CHXjoy6kHol2Lo-kFJmoNBvNBkKFpFJ3oD6ejTse7718r7WSUzeh4R_vV9QNEHNPucxpL3Yhm_EuYIMV-cfA_N58dA1YAjcZtlEM8PsFwDQGv5vTndGkQ_co0acuDBgXZsJ6xCaNvpgrx_ftpzlaA27PknKK6rrvTRSuxKP4Jn3GIB0nBa6uXUfMvlkUBvepwXooXO5XAlWRTa3J6ys2KkOVkDKMN-jdSv-K3_rLXxxb3Q"
```

```shell
export WRONG_AUDIENCE_TOKEN="eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJhdWQiOiJ3cm9uZy1hdWRpZW5jZS5jb20iLCJleHAiOjE5MTUyNTc5NzksImlhdCI6MTU5OTY4ODQ1OSwiaXNzIjoidGVzdGluZ0BzZWN1cmUuaXN0aW8uaW8iLCJzdWIiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyJ9.ELpcxqd9T6ZmdQ6Ilak26yiizi205az0hSmNnVALV_pCfbM2frMaNnESotFILe77Z3u-HeiVTomrgf-onAUW0bZJ3HWi_iPJZHQ5bq2-rvf5_E4IAr_p39argaVriCVRiPnBtuwzzptWOpS-wBe5uDr3V6otX7PcAH9s1FCCMnTxiEJzn_taTYLtpctoHDsNfw5RfDvVZbsuyEX1Ea0UUx1-VZrVV7n5_jXmo8yoHQ5DH6QDgpGJlYMebgLgoG7pL8N2oz7VLQQSO0x-rIdv_Icdee04fnrOzs01YdGWqfEfDjGbAjrqsq0e1HXhHF0AJNcsuNcFcQQv4nePyuKVyg"
```

To view the token's claims use the following command:

```shell
echo $VALID_TOKEN | cut -d '.' -f2 | base64 --decode | awk '{print $0}'
base64: invalid input
{"aud":"boutiquestore.com","exp":1914365365,"iat":1599005365,"iss":"testing@secure.istio.io","sub":"user1@xyz.com"}
```

## Configure JWT-based authentication

Create a `RequestAuthentication` resource and apply it to the Istio ingress gateway (mesh scope).
This will configure the mesh for authentication.

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: online-boutique-request-authn
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.9/security/tools/jwt/samples/jwks.json"
```

```shell
kubectl apply -n istio-system -f jwt-request-authentication.yaml
```

Note that the RequestAuthentication resource uses `jwtRules` to configure the
authentication mechanism. This resource configures the JSON Web Key (JWK) URI. 
The Istio control plane will use the JWK to fetch the public key set used to 
validate the user’s JWT and pass it to the proxies. 
Additionally, the Istio control plane periodically refreshes the public keys 
and automatically updates the proxies (in this case, ingress gateway) when 
the authentication provider’s public key changes. 
A demo JWK public key URI is used that will match the private key that signed the
demo tokens (below) -- https://istio.io/latest/docs/tasks/security/authentication/authn-policy/#end-user-authentication

You can set the `fromHeaders` field in the RequestAuthentication resource to 
configure the header from the incoming HTTP request, which should be used to 
the JWT. By default, the Authorization HTTP header is used with the “Bearer” 
prefix to extract the JWT.

You can provide additional JWT identity providers by adding multiple entries in 
the `jwtRules` array. This is useful if you’re migrating identity providers or 
allowing multiple JWT identity issuers.

## Verify JWT authentication

### Use the valid token.

```shell
curl -v -H "Authorization: Bearer $VALID_TOKEN" http://$INGRESS:80/_healthz
*   Trying 192.168.1.41:80...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 192.168.1.41 (192.168.1.41) port 80 (#0)
> GET /_healthz HTTP/1.1
> Host: 192.168.1.41
> User-Agent: curl/7.77.0
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJhdWQiOiJib3V0aXF1ZXN0b3JlLmNvbSIsImV4cCI6MTkxNDM2NTM2NSwiaWF0IjoxNTk5MDA1MzY1LCJpc3MiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyIsInN1YiI6InVzZXIxQHh5ei5jb20ifQ.pZHGJgEP0qsJ1pSBPU6nwpMETf3MqlWx-T0PED0BLb5aTnbXWklIzHV4JHgMk6Td4thI4cC4_mCvrFlgm1uqzO2U_DEdNOReNMGNnDSQ87hWEtOQxn50c83dVq3-w4FUrKBjS1lYhiabKWVfr3BC_Bmwefl1TVhgP_iNuQGsWsj_Tcl4TWP93yS93DazphAl0uIOw3WvmJSfhnaZ7cEIk0PzGmsLq96FLbGatZX1DL8OKhv2Hz-iN0V6KG_5GjjEdD7LVSprw4oedtwbahDVVuxh0due1DImXLhH9oH1_RkntrD98JlKYoNzG1-7apSV0xw5mlE8L2HXDo3TKE33WA
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< set-cookie: shop_session-id=9a48f65d-9167-46ec-b2af-b580bb42ff86; Max-Age=172800
< date: Sat, 18 Jun 2022 17:34:20 GMT
< content-length: 2
< content-type: text/plain; charset=utf-8
< x-envoy-upstream-service-time: 3
< server: istio-envoy
<
{ [2 bytes data]
100     2  100     2    0     0     49      0 --:--:-- --:--:-- --:--:--    51ok
* Connection #0 to host 192.168.1.41 left intact
```
The ingress gateway responds with HTTP response code 200, and the message “ok” 
is received from the frontend service. This implies that the ingress gateway 
forwarded the request to the upstream frontend service after validating that 
the JWT is issued and signed by the configured issuer.

### Verify over HTTPS

Make sure to set the following env:
```shell
export INGRESS_IP=$INGRESS
```

```shell
curl -H "Authorization: Bearer $VALID_TOKEN" \
    -H "Host: marketplace.boutiquestore.com" \
    --cacert ../tls/online-boutique-tls-credential-root.crt \
    --resolve "marketplace.boutiquestore.com:443:$INGRESS_IP" \
    "https://marketplace.boutiquestore.com:443/_healthz"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100     2  100     2    0     0     71      0 --:--:-- --:--:-- --:--:--    74ok
```

### Use an invalid token

```shell
curl -v -H "Authorization: Bearer $INVALID_TOKEN" http://$INGRESS:80/_healthz
*   Trying 192.168.1.41:80...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 192.168.1.41 (192.168.1.41) port 80 (#0)
> GET /_healthz HTTP/1.1
> Host: 192.168.1.41
> User-Agent: curl/7.77.0
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJhdWQiOiJib3V0aXF1ZXN0b3JlLmNvbSIsImV4cCI6MTkxNDM2NTI1MSwiaWF0IjoxNTk5MDA1MjUxLCJpc3MiOiJ3cm9uZy1pc3N1c2VyQHNlY3VyZS5pc3Rpby5pbyIsInN1YiI6InVzZXIxQHh5ei5jb20ifQ.SnQPGlkV66Q61zR8uZAzEhyPfynNmV_MGzvjnkxZhVv-elrKu7Wq50tj4SKGyjTJPR2-YFd_-p3eN4VveCH5NB3LjgyOliMQjnxSTN92CHXjoy6kHol2Lo-kFJmoNBvNBkKFpFJ3oD6ejTse7718r7WSUzeh4R_vV9QNEHNPucxpL3Yhm_EuYIMV-cfA_N58dA1YAjcZtlEM8PsFwDQGv5vTndGkQ_co0acuDBgXZsJ6xCaNvpgrx_ftpzlaA27PknKK6rrvTRSuxKP4Jn3GIB0nBa6uXUfMvlkUBvepwXooXO5XAlWRTa3J6ys2KkOVkDKMN-jdSv-K3_rLXxxb3Q
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 401 Unauthorized
< content-length: 28
< content-type: text/plain
< date: Sat, 18 Jun 2022 17:42:36 GMT
< server: istio-envoy
<
{ [28 bytes data]
100    28  100    28    0     0   1257      0 --:--:-- --:--:-- --:--:--  1333Jwt issuer is not configured
* Connection #0 to host 192.168.1.41 left intact
```

The ingress gateway responds with HTTP response code 401 Unauthorized, 
and the message “Jwt issuer is not configured”.

## Configure an Authorization policy at the ingress gateway
Fine-grained access control can be configured based on the various attributes of the request (Authorization policy).

Create an `AuthorizationPolicy` resource and apply it to the Istio ingress gateway (mesh scope).

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: online-boutique
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"]
```
```shell
kubectl apply -n istio-system -f deny-no-or-invalid-jwt-access-policy.yaml
```

Basically, this policy resource denies access to requests that have no JWTs or invalid JWTs.

This authorization policy is not scoped to any port or specific path or host, which means that the policy affects all
ports and host/paths exposed via the ingress gateway.

## Verify the authorization policy

```shell
curl -v -H "Authorization: Bearer $VALID_TOKEN" http://$INGRESS:80/_healthz
*   Trying 192.168.1.41:80...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 192.168.1.41 (192.168.1.41) port 80 (#0)
> GET /_healthz HTTP/1.1
> Host: 192.168.1.41
> User-Agent: curl/7.77.0
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJhdWQiOiJib3V0aXF1ZXN0b3JlLmNvbSIsImV4cCI6MTkxNDM2NTM2NSwiaWF0IjoxNTk5MDA1MzY1LCJpc3MiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyIsInN1YiI6InVzZXIxQHh5ei5jb20ifQ.pZHGJgEP0qsJ1pSBPU6nwpMETf3MqlWx-T0PED0BLb5aTnbXWklIzHV4JHgMk6Td4thI4cC4_mCvrFlgm1uqzO2U_DEdNOReNMGNnDSQ87hWEtOQxn50c83dVq3-w4FUrKBjS1lYhiabKWVfr3BC_Bmwefl1TVhgP_iNuQGsWsj_Tcl4TWP93yS93DazphAl0uIOw3WvmJSfhnaZ7cEIk0PzGmsLq96FLbGatZX1DL8OKhv2Hz-iN0V6KG_5GjjEdD7LVSprw4oedtwbahDVVuxh0due1DImXLhH9oH1_RkntrD98JlKYoNzG1-7apSV0xw5mlE8L2HXDo3TKE33WA
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< set-cookie: shop_session-id=9502c68d-c670-4894-9067-8bffb070c567; Max-Age=172800
< date: Sun, 19 Jun 2022 14:42:36 GMT
< content-length: 2
< content-type: text/plain; charset=utf-8
< x-envoy-upstream-service-time: 26
< server: istio-envoy
<
{ [2 bytes data]
100     2  100     2    0     0      5      0 --:--:-- --:--:-- --:--:--     5ok
* Connection #0 to host 192.168.1.41 left intact
```

The result is an HTTP response code 200 OK because the request contains a valid JWT.

Now issue the same command without the JWT.
```shell
curl -v http://$INGRESS:80/_healthz
*   Trying 192.168.1.41:80...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 192.168.1.41 (192.168.1.41) port 80 (#0)
> GET /_healthz HTTP/1.1
> Host: 192.168.1.41
> User-Agent: curl/7.77.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 403 Forbidden
< content-length: 19
< content-type: text/plain
< date: Sun, 19 Jun 2022 14:45:56 GMT
< server: istio-envoy
<
{ [19 bytes data]
100    19  100    19    0     0    655      0 --:--:-- --:--:-- --:--:--   678RBAC: access denied
* Connection #0 to host 192.168.1.41 left intact
```
The request is now denied with an HTTP response code 403 Forbidden and the message “RBAC: access denied”.

When issuing the same command however with an invalid token we get an HTTP response code 401 Unauthorized.
```shell
curl -v -H "Authorization: Bearer $INVALID_TOKEN" http://$INGRESS:80/_healthz
*   Trying 192.168.1.41:80...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 192.168.1.41 (192.168.1.41) port 80 (#0)
> GET /_healthz HTTP/1.1
> Host: 192.168.1.41
> User-Agent: curl/7.77.0
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJhdWQiOiJib3V0aXF1ZXN0b3JlLmNvbSIsImV4cCI6MTkxNDM2NTI1MSwiaWF0IjoxNTk5MDA1MjUxLCJpc3MiOiJ3cm9uZy1pc3N1c2VyQHNlY3VyZS5pc3Rpby5pbyIsInN1YiI6InVzZXIxQHh5ei5jb20ifQ.SnQPGlkV66Q61zR8uZAzEhyPfynNmV_MGzvjnkxZhVv-elrKu7Wq50tj4SKGyjTJPR2-YFd_-p3eN4VveCH5NB3LjgyOliMQjnxSTN92CHXjoy6kHol2Lo-kFJmoNBvNBkKFpFJ3oD6ejTse7718r7WSUzeh4R_vV9QNEHNPucxpL3Yhm_EuYIMV-cfA_N58dA1YAjcZtlEM8PsFwDQGv5vTndGkQ_co0acuDBgXZsJ6xCaNvpgrx_ftpzlaA27PknKK6rrvTRSuxKP4Jn3GIB0nBa6uXUfMvlkUBvepwXooXO5XAlWRTa3J6ys2KkOVkDKMN-jdSv-K3_rLXxxb3Q
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 401 Unauthorized
< content-length: 28
< content-type: text/plain
< date: Sun, 19 Jun 2022 14:54:33 GMT
< server: istio-envoy
<
{ [28 bytes data]
100    28  100    28    0     0   1147      0 --:--:-- --:--:-- --:--:--  1217Jwt issuer is not configured
* Connection #0 to host 192.168.1.41 left intact
```
The resource configured above only allows a valid principal to access the online boutique store; 
it does not perform any additional validation on the JWT itself. 
By default, the RequestAuthentication resource validates signature, issuer, and expiration 
but does no validation on the targeted audience for the issued JWT.
You can perform stricter access-control enforcement by adding conditions to the authorization policy 
resource (see next).

## Configure an audience-scoped authorization policy

Update the Authorization policy resource with the following:
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: online-boutique
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: ALLOW
  rules:
  - when:
    - key: "request.auth.audiences"
      values: ["boutiquestore.com"]
```

```shell
kubectl apply -n istio-system -f allow-audience-access-policy.yaml
```
The new resource is an ALLOW action policy, which will only allow requests that match the rules defined in the resource.
This means that any request not matching the rule will be denied.

## Verify audience-scope authorization policy

```shell
curl -v -H "Authorization: Bearer $WRONG_AUDIENCE_TOKEN" http://$INGRESS:80/_healthz
*   Trying 192.168.1.41:80...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 192.168.1.41 (192.168.1.41) port 80 (#0)
> GET /_healthz HTTP/1.1
> Host: 192.168.1.41
> User-Agent: curl/7.77.0
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJhdWQiOiJ3cm9uZy1hdWRpZW5jZS5jb20iLCJleHAiOjE5MTUyNTc5NzksImlhdCI6MTU5OTY4ODQ1OSwiaXNzIjoidGVzdGluZ0BzZWN1cmUuaXN0aW8uaW8iLCJzdWIiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyJ9.ELpcxqd9T6ZmdQ6Ilak26yiizi205az0hSmNnVALV_pCfbM2frMaNnESotFILe77Z3u-HeiVTomrgf-onAUW0bZJ3HWi_iPJZHQ5bq2-rvf5_E4IAr_p39argaVriCVRiPnBtuwzzptWOpS-wBe5uDr3V6otX7PcAH9s1FCCMnTxiEJzn_taTYLtpctoHDsNfw5RfDvVZbsuyEX1Ea0UUx1-VZrVV7n5_jXmo8yoHQ5DH6QDgpGJlYMebgLgoG7pL8N2oz7VLQQSO0x-rIdv_Icdee04fnrOzs01YdGWqfEfDjGbAjrqsq0e1HXhHF0AJNcsuNcFcQQv4nePyuKVyg
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 403 Forbidden
< content-length: 19
< content-type: text/plain
< date: Sun, 19 Jun 2022 15:18:03 GMT
< server: istio-envoy
<
{ [19 bytes data]
100    19  100    19    0     0    572      0 --:--:-- --:--:-- --:--:--   612RBAC: access denied
* Connection #0 to host 192.168.1.41 left intact
```
The response received is an HTTP response code 403 forbidden, indicating that the Istio ingress gateway denied this request due to the incorrect audience claim in the JWT.

## Configure an audience and path-scoped authorization policy

Update the Authorization policy resource with the following:
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: online-boutique
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY
  rules:
  - when:
    - key: "request.auth.audiences"
      notValues: ["boutiquestore.com"]
    to:
    - operation:
        paths: ["/cart"]
```

```shell
kubectl apply -n istio-system -f deny-audience-and-path-access-policy.yaml
```

This resource is a DENY policy resource configured to deny requests if the request authentication 
audience is not set to `boutiquestore.com`.
The rule also adds a “to” section, which filters down the requests that this policy applies to. 
In this case, this policy is only applicable to requests to the path `/cart`.

## Verify audience and path-scoped authorization policy

Since the resource is scoped to the /cart path, any requests to `/_healthz` should work without 
any JWTs.

```shell
curl -v http://$INGRESS:80/_healthz
*   Trying 192.168.1.41:80...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 192.168.1.41 (192.168.1.41) port 80 (#0)
> GET /_healthz HTTP/1.1
> Host: 192.168.1.41
> User-Agent: curl/7.77.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< set-cookie: shop_session-id=f27c85f2-1740-49f8-8770-a299c35e15e6; Max-Age=172800
< date: Sun, 19 Jun 2022 15:53:08 GMT
< content-length: 2
< content-type: text/plain; charset=utf-8
< x-envoy-upstream-service-time: 33
< server: istio-envoy
<
{ [2 bytes data]
100     2  100     2    0     0      4      0 --:--:-- --:--:-- --:--:--     4ok
* Connection #0 to host 192.168.1.41 left intact
```

Now access the `/cart` path without a JWT.

```shell
curl -v http://$INGRESS:80/cart
*   Trying 192.168.1.41:80...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 192.168.1.41 (192.168.1.41) port 80 (#0)
> GET /cart HTTP/1.1
> Host: 192.168.1.41
> User-Agent: curl/7.77.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 403 Forbidden
< content-length: 19
< content-type: text/plain
< date: Sun, 19 Jun 2022 15:54:40 GMT
< server: istio-envoy
<
{ [19 bytes data]
100    19  100    19    0     0     69      0 --:--:-- --:--:-- --:--:--    69RBAC: access denied
* Connection #0 to host 192.168.1.41 left intact
```
The request is rejected with an HTTP response code 403 Forbidden and the message “RBAC: access denied.”

Now add a valid JWT to the request.

```shell
curl -v -sS -H "Authorization: Bearer $VALID_TOKEN" --write-out "%{http_code}" -o /dev/null "http://$INGRESS:80/cart"
200*   Trying 192.168.1.41:80...
* Connected to 192.168.1.41 (192.168.1.41) port 80 (#0)
> GET /cart HTTP/1.1
> Host: 192.168.1.41
> User-Agent: curl/7.77.0
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJhdWQiOiJib3V0aXF1ZXN0b3JlLmNvbSIsImV4cCI6MTkxNDM2NTM2NSwiaWF0IjoxNTk5MDA1MzY1LCJpc3MiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyIsInN1YiI6InVzZXIxQHh5ei5jb20ifQ.pZHGJgEP0qsJ1pSBPU6nwpMETf3MqlWx-T0PED0BLb5aTnbXWklIzHV4JHgMk6Td4thI4cC4_mCvrFlgm1uqzO2U_DEdNOReNMGNnDSQ87hWEtOQxn50c83dVq3-w4FUrKBjS1lYhiabKWVfr3BC_Bmwefl1TVhgP_iNuQGsWsj_Tcl4TWP93yS93DazphAl0uIOw3WvmJSfhnaZ7cEIk0PzGmsLq96FLbGatZX1DL8OKhv2Hz-iN0V6KG_5GjjEdD7LVSprw4oedtwbahDVVuxh0due1DImXLhH9oH1_RkntrD98JlKYoNzG1-7apSV0xw5mlE8L2HXDo3TKE33WA
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< set-cookie: shop_session-id=7b5f5163-7cc4-41bf-8353-f4a3de2610cc; Max-Age=172800
< date: Sun, 19 Jun 2022 15:57:19 GMT
< content-type: text/html; charset=utf-8
< x-envoy-upstream-service-time: 59
< server: istio-envoy
< transfer-encoding: chunked
<
{ [1198 bytes data]
* Connection #0 to host 192.168.1.41 left intact
```

## <a id="milestone4"></a> Configure audience, path and port scoped authorization policy

Update the resource to scope it to only affect requests to HTTPS port 443 and not port 80.
Verify that traffic to path `/cart` on port 80 is allowed without a JWT but rejected on port 443.

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: online-boutique
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY
  rules:
  - when:
    - key: "request.auth.audiences"
      notValues: ["boutiquestore.com"]
    to:
    - operation:
        ports: ["443"]
        paths: ["/cart"]
```

```shell
kubectl apply -n istio-system -f deny-audience-path-and-port-access-policy.yaml
```

## Verify audience, path and port scope authorization policy

```shell
curl -v -sS --write-out "%{http_code}" -o /dev/null http://$INGRESS:80/cart
200*   Trying 192.168.1.41:80...
* Connected to 192.168.1.41 (192.168.1.41) port 80 (#0)
> GET /cart HTTP/1.1
> Host: 192.168.1.41
> User-Agent: curl/7.77.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< set-cookie: shop_session-id=f8fc207e-c428-4234-931d-8ca266c6d181; Max-Age=172800
< date: Sun, 19 Jun 2022 20:09:52 GMT
< content-type: text/html; charset=utf-8
< x-envoy-upstream-service-time: 16
< server: istio-envoy
< transfer-encoding: chunked
<
{ [1198 bytes data]
* Connection #0 to host 192.168.1.41 left intact
```


**Note**: with my deployed version of Istio *1.10* on __microk8s__ I have noticed that
the `ports` field does not apply the policy to requests with port __443__; this is possibly due to the way I had
to enable and configure the __metallb__ loadbalancer on __microk8s__.

```shell
$ microk8s.kubectl get svc -n istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                                                                      AGE
istiod                 ClusterIP      10.152.183.59   <none>         15010/TCP,15012/TCP,443/TCP,15014/TCP                                        53d
istio-egressgateway    ClusterIP      10.152.183.69   <none>         80/TCP,443/TCP                                                               53d
istio-ingressgateway   LoadBalancer   10.152.183.48   192.168.1.41   15021:32102/TCP,80:32394/TCP,443:30543/TCP,31400:31766/TCP,15443:30332/TCP   53d
```

(:warning:) The following HTTPS request succeeds with HTTP 200 OK response code rather than
HTTP 403 Forbidden.

```shell
curl -sS -H "Host: marketplace.boutiquestore.com" \
    --cacert ../tls/online-boutique-tls-credential-root.crt \
    --resolve "marketplace.boutiquestore.com:443:$INGRESS_IP" \
    --write-out "%{http_code}" -o /dev/null \
    "https://marketplace.boutiquestore.com:443/cart"
200
```
