# Encrypting End-User Traffic

## Create Kubernetes TLS credentials

Firstly, we need to create the root CA and server key and certificate.

> If running on Windows Git Bash, set the following before running the script   `export MSYS_NO_PATHCONV=1`

```shell
./generate_tls_credentials.sh boutiquestore '.com' marketplace online-boutique-tls-credential
```

This script will create the server private key and public key, signed by the self-signed root CA.

You should find a file that is created with the following name, `online-boutique-tls-credential.yaml`.

Create a Kubernetes secret with the generated server private and public keys.

```shell
kubectl apply -f online-boutique-tls-credential.yaml
```

## Configure the TLS Gateway resource

Enhance the Istio ingress Gateway by exposing port 443 and have it perform TLS termination for
HTTPS traffic.

Add the following to the `gateway-http.yaml` resource definition file, under the `spec.servers` section.

```yaml
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: online-boutique-tls-credential
    hosts:
      - "marketplace.boutiquestore.com"

```

TLS mode `SIMPLE` means that no client certificate verification is performed, hence
no mutual TLS (mTLS).

The `credentialName` should refer to the Kubernetes TLS secret of the same name created above.

The `host` field should match the TLS certificate common name. The ingress gateway only
offers this certificate to a client that initiates a TLS handshake with the same hostname.

HTTPS traffic will only be matched if the requests are for the __server name indication (SNI)*__
and the HTTP Host header matches the hostname.

> __*__ Server Name Indication is an extension to the Transport Layer Security computer networking protocol by which a client
> indicates which hostname it is attempting to connect to at the start of the handshaking process.
> *(Wikipedia)*

## Verify HTTPS traffic

Set up the running environment for executing cUrl commands.

```shell
. ./setup-env.sh
export INGRESS_IP=$INGRESS
```

```shell
$ curl -H "Host: marketplace.boutiquestore.com" --cacert tls/online-boutique-tls-credential-root.crt \
  --resolve "marketplace.boutiquestore.com:443:$INGRESS_IP" "https://marketplace.boutiquestore.com:443/_healthz"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100     2  100     2    0     0     28      0 --:--:-- --:--:-- --:--:--    29ok

```

We need to point to the self-signed root CA certificate to verify the server certificate.
We also need to resolve the domain name `marketplace.boutiquestore.com` to the ingress
IP address.

## Verify TLS server certificate information

Verify that Istio ingress gateway presents the correct TLS certificate.

```shell
$ openssl s_client -showcerts -no_tls1_3 -connect $INGRESS:443 -servername marketplace.boutiquestore.com -CAfile tls/online-boutique-tls-credential-root.crt
```
```shell
CONNECTED(00000138)
---
Certificate chain
 0 s:CN = marketplace.boutiquestore.com, O = boutiquestore
   i:O = boutiquestore Inc., CN = boutiquestore.com
-----BEGIN CERTIFICATE-----
MIIC7TCCAdUCAQAwDQYJKoZIhvcNAQELBQAwOTEbMBkGA1UECgwSYm91dGlxdWVz
dG9yZSBJbmMuMRowGAYDVQQDDBFib3V0aXF1ZXN0b3JlLmNvbTAeFw0yMjA2MDQx
ODE4MjBaFw0yMzA2MDQxODE4MjBaMEAxJjAkBgNVBAMMHW1hcmtldHBsYWNlLmJv
dXRpcXVlc3RvcmUuY29tMRYwFAYDVQQKDA1ib3V0aXF1ZXN0b3JlMIIBIjANBgkq
hkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAuawDbo/TfMsSpxB9BEm2ZMrJR885zHED
HqVL3zXBHGJlrxRjygaSSpk/acFkBqGtAid1Fenwgg1ewdATZQjc2xxndZet3XGf
bhMI6XkQKcL5d9KiuqRuayMP13OpxErXQDd5OkvvmxRYfFyR8xRMo1E1IVYumNbr
i0+MWwe56PZzE9vWbPC60V/2y97yOeWvGiVnivNeRSxBzARn+88xwsKqda7G3wCp
7LmxRcrLs6+Q1AfdmURP0DJUqZXs+VoCWIDgMKIxXLSTg0jyJAOaqXy+FaWYK6ge
KUOOJMybbJNQ+hdZx4KyDeV5Dj/X6lTwhpSiBkPv2SHLpQwNhHIE5QIDAQABMA0G
CSqGSIb3DQEBCwUAA4IBAQBzpYfTQUwnVRZVKThCwPOE4+DZt/qZOSM2S6eu5OLX
7PFLAwdMnNItD2bvCW9rybVM9Pyvdbh9QBCdYHsrdVpUfWURibqok8ju40GQc9zd
cD4J3AATL5Gh3mN4bouD7sadla+sWMB/Yo8cY/geaSLOADJgjLV2TGpkgw07uih2
dqMeEC7TEfpucwU5ifKCJobeGn0+2fjmudZiyjlRcMR3gNT0zVVyDtM5M/AuVKX/
l7FAir0GTdL3qO36xU8hg/nwjy9R13cU0Fevog4ebaRWzxlCCwz594wpKr1H+HrG
mZaHEmp1A2j7XWYPMRbCnkRBsgHPBieHXlOoLyeY2/9v
-----END CERTIFICATE-----
---
Server certificate
subject=CN = marketplace.boutiquestore.com, O = boutiquestore

issuer=O = boutiquestore Inc., CN = boutiquestore.com

---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1400 bytes and written 317 bytes
Verification: OK
---
New, TLSv1.2, Cipher is ECDHE-RSA-CHACHA20-POLY1305
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-CHACHA20-POLY1305
    Session-ID: 6F511162FCC9D5C16F912895DD54EAED9452A931D56FD1EBE42B500BFAED12C0
    Session-ID-ctx:
    Master-Key: 3367261A9D6CEB7059F5FEDF797AFF7FB27DD0948EBE82E05412CCF079B0361C485905B065D888D3FF015B088ABFC14C
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 7200 (seconds)
    TLS session ticket:
    0000 - 8c dc f9 ff bf 4b 92 5e-2f b8 13 a5 cd dc 57 69   .....K.^/.....Wi
    0010 - 84 56 b8 b8 cd 48 36 5f-2c 25 3f 3d ff 0e b4 f0   .V...H6_,%?=....
    0020 - 39 d7 33 48 37 80 4d ff-ce c1 e9 3d a3 cc ac 75   9.3H7.M....=...u
    0030 - 81 5b b7 23 88 f1 05 fc-99 5e e6 55 31 e1 40 4b   .[.#.....^.U1.@K
    0040 - a5 7b 42 58 57 e3 14 38-67 9f f1 4d a0 89 47 10   .{BXW..8g..M..G.
    0050 - 09 ff e3 64 13 4f b5 3a-2c 55 88 c1 53 fe f6 7d   ...d.O.:,U..S..}
    0060 - be aa 0a b4 ed 13 ac 82-cc 98 8d 05 1e 27 ca 3e   .............'.>
    0070 - 5e 37 be b7 99 70 94 8d-a7 0e 59 cc ef 34 8c 32   ^7...p....Y..4.2
    0080 - 48 3f 08 5c 21 3b 13 58-bb 98 18 32 41 2b 4d a1   H?.\!;.X...2A+M.
    0090 - c3 d1 e6 0b d6 db 81 e2-0a 5d 0c d0 80 ef f4 6f   .........].....o
    00a0 - d1 42 ab c1 1c aa e2 df-37 45 e5 0b 86 f0 d9 d4   .B......7E......
    00b0 - 70 82 8b 4b df be c2 d9-03 81 ab 07 f4 46 2a 51   p..K.........F*Q

    Start Time: 1654418707
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: yes
---
```

When the `servername` option is present, then the OpenSSL client sets the SNI header
correctly.

Without the `servername` option we get the following:

```shell
$ openssl s_client -showcerts -no_tls1_3 -connect $INGRESS:443 -CAfile tls/online-boutique-tls-credential-root.crt
CONNECTED(00000154)
---
no peer certificate available
---
No client certificate CA names sent
---
SSL handshake has read 0 bytes and written 194 bytes
Verification: OK
---
New, (NONE), Cipher is (NONE)
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : 0000
    Session-ID:
    Session-ID-ctx:
    Master-Key:
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    Start Time: 1654418938
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
---
write:errno=10054
```

Without the CA file we get the following:

```shell
$ openssl s_client -showcerts -no_tls1_3 -connect $INGRESS:443 -servername marketplace.boutiquestore.com
CONNECTED(00000138)
---
Certificate chain
 0 s:CN = marketplace.boutiquestore.com, O = boutiquestore
   i:O = boutiquestore Inc., CN = boutiquestore.com
-----BEGIN CERTIFICATE-----
MIIC7TCCAdUCAQAwDQYJKoZIhvcNAQELBQAwOTEbMBkGA1UECgwSYm91dGlxdWVz
dG9yZSBJbmMuMRowGAYDVQQDDBFib3V0aXF1ZXN0b3JlLmNvbTAeFw0yMjA2MDQx
ODE4MjBaFw0yMzA2MDQxODE4MjBaMEAxJjAkBgNVBAMMHW1hcmtldHBsYWNlLmJv
dXRpcXVlc3RvcmUuY29tMRYwFAYDVQQKDA1ib3V0aXF1ZXN0b3JlMIIBIjANBgkq
hkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAuawDbo/TfMsSpxB9BEm2ZMrJR885zHED
HqVL3zXBHGJlrxRjygaSSpk/acFkBqGtAid1Fenwgg1ewdATZQjc2xxndZet3XGf
bhMI6XkQKcL5d9KiuqRuayMP13OpxErXQDd5OkvvmxRYfFyR8xRMo1E1IVYumNbr
i0+MWwe56PZzE9vWbPC60V/2y97yOeWvGiVnivNeRSxBzARn+88xwsKqda7G3wCp
7LmxRcrLs6+Q1AfdmURP0DJUqZXs+VoCWIDgMKIxXLSTg0jyJAOaqXy+FaWYK6ge
KUOOJMybbJNQ+hdZx4KyDeV5Dj/X6lTwhpSiBkPv2SHLpQwNhHIE5QIDAQABMA0G
CSqGSIb3DQEBCwUAA4IBAQBzpYfTQUwnVRZVKThCwPOE4+DZt/qZOSM2S6eu5OLX
7PFLAwdMnNItD2bvCW9rybVM9Pyvdbh9QBCdYHsrdVpUfWURibqok8ju40GQc9zd
cD4J3AATL5Gh3mN4bouD7sadla+sWMB/Yo8cY/geaSLOADJgjLV2TGpkgw07uih2
dqMeEC7TEfpucwU5ifKCJobeGn0+2fjmudZiyjlRcMR3gNT0zVVyDtM5M/AuVKX/
l7FAir0GTdL3qO36xU8hg/nwjy9R13cU0Fevog4ebaRWzxlCCwz594wpKr1H+HrG
mZaHEmp1A2j7XWYPMRbCnkRBsgHPBieHXlOoLyeY2/9v
-----END CERTIFICATE-----
---
Server certificate
subject=CN = marketplace.boutiquestore.com, O = boutiquestore

issuer=O = boutiquestore Inc., CN = boutiquestore.com

---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1400 bytes and written 317 bytes
Verification error: unable to verify the first certificate
---
New, TLSv1.2, Cipher is ECDHE-RSA-CHACHA20-POLY1305
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-CHACHA20-POLY1305
    Session-ID: 2EA7508B71B05D42A4C779E009B47F4C65E51CA71DCECFAC83107E9E6496BADB
    Session-ID-ctx:
    Master-Key: 7331C6412D7B1BEEFB7FEE7DE088D46929E66C66D0380F254F82092F11DEFC61497AC2FE4D4C625BDA1EB4387C9D827E
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 7200 (seconds)
    TLS session ticket:
    0000 - 8c dc f9 ff bf 4b 92 5e-2f b8 13 a5 cd dc 57 69   .....K.^/.....Wi
    0010 - 7c 1f 68 15 a7 66 e5 7a-31 25 4c 32 92 28 fa 20   |.h..f.z1%L2.(.
    0020 - bb a1 84 98 f1 bf a8 c1-1e 1a 03 bc 6f 79 32 31   ............oy21
    0030 - 61 a5 14 fe 37 94 17 fe-e9 e7 08 8e b8 30 78 30   a...7........0x0
    0040 - 58 22 d1 7b e4 71 fc e8-29 27 0e a6 a3 8d 33 ea   X".{.q..)'....3.
    0050 - 89 48 39 b0 a3 6e fe 42-f8 c3 3f d0 e3 0d fc 9e   .H9..n.B..?.....
    0060 - cd de ee 73 4c fc 5d 30-80 46 27 dc fe 1f 31 8c   ...sL.]0.F'...1.
    0070 - e6 30 ee 35 89 35 25 c7-b3 a7 e7 87 d5 8c 77 23   .0.5.5%.......w#
    0080 - 63 b1 e7 87 1a 01 2a 76-df 6e 5c 6b ea 89 d5 36   c.....*v.n\k...6
    0090 - d1 af fa cc 8a 24 4b 1c-f8 da 54 69 7b 31 f4 af   .....$K...Ti{1..
    00a0 - be 35 c3 94 b9 b3 82 38-f5 71 e7 b9 24 7a bd 9d   .5.....8.q..$z..
    00b0 - 55 83 4d a5 91 f2 ef ce-62 29 f7 ad c8 c3 40 93   U.M.....b)....@.

    Start Time: 1654419195
    Timeout   : 7200 (sec)
    Verify return code: 21 (unable to verify the first certificate)
    Extended master secret: yes
---
```

