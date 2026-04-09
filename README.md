# Cilium GatewayAPI Demos

A self-contained demo exploring various routing scenarios through the [Cilium Gateway API](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/) using a local [KIND](https://kind.sigs.k8s.io/) cluster (with a focus on HTTP/2).

## Table of Contents

- [Table of Contents](#table-of-contents)
- [Context](#context)
- [Scenarios](#scenarios)
  - [HTTP](#http)
  - [HTTPS (TLS Termination)](#https-tls-termination)
  - [TLS Passthrough](#tls-passthrough)
  - [gRPC](#grpc)
  - [Not Implemented](#not-implemented)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
  - [1. Add the Cilium Helm Repository](#1-add-the-cilium-helm-repository)
  - [2. Create the KIND Cluster](#2-create-the-kind-cluster)
  - [3. Install the Gateway API CRDs](#3-install-the-gateway-api-crds)
  - [4. Install Cilium](#4-install-cilium)
  - [5. Wait for Cilium to Be Ready](#5-wait-for-cilium-to-be-ready)
  - [6. Create the TLS Certificate for the HTTPS Gateway Listener](#6-create-the-tls-certificate-for-the-https-gateway-listener)
  - [7. Deploy echo-app and the Gateways](#7-deploy-echo-app-and-the-gateways)
  - [8. Wait for the Deployment](#8-wait-for-the-deployment)
- [Testing](#testing)
  - [HTTP](#http-1)
  - [HTTPS (TLS Termination)](#https-tls-termination-1)
  - [TLS Passthrough](#tls-passthrough-1)
  - [gRPC](#grpc-1)
- [Teardown](#teardown)
- [Repository Structure](#repository-structure)
- [License](#license)

## Context

The Cilium Gateway API implementation (backed by Envoy) supports HTTP/2 via two complementary Helm settings:

- **`gatewayAPI.enableAlpn`**: Envoy advertises `h2` and `http/1.1` via ALPN on all Gateway listeners and selects HTTP/2 when the client supports it (downstream leg).
- **`gatewayAPI.enableAppProtocol`**: Enables Backend Protocol Selection ([GEP-1911](https://gateway-api.sigs.k8s.io/geps/gep-1911/)). Cilium reads the `appProtocol` field on Service ports to determine the protocol used for upstream connections (Gateway → backend leg).

> **Note:** In the Cilium Helm chart, `enableAlpn: true` implicitly forces `enableAppProtocol` on as well — regardless of how `enableAppProtocol` is set. Setting `enableAppProtocol: false` therefore has no effect when `enableAlpn: true`.

The backend used throughout this demo is [`echo-app`](https://github.com/PhilipSchmid/echo-app), a Go application that echoes back request details as JSON — including the HTTP version seen by the pod — making it easy to observe protocol behaviour at every hop.

## Scenarios

Scenario titles use **downstream** (client → Gateway) and **upstream** (Gateway → backend) to describe each leg of the connection.

### HTTP

#### Scenario A — HTTP/1.1 downstream, HTTP/1.1 upstream ✅

The baseline: a plain HTTP/1.1 request through the Gateway to the backend. No `appProtocol` annotation is needed on the Service. ALPN-based HTTP/2 negotiation is not possible here because the Gateway listener is plain HTTP (no TLS). Scenario B introduces h2c on the upstream leg; Scenarios D and E introduce full HTTP/2 on both legs using a TLS listener with ALPN.

```
curl → Cilium Gateway (HTTP/1.1) → echo-app Service (HTTP/1.1) → echo-app Pod
```

#### Scenario B — HTTP/1.1 downstream, HTTP/2 upstream (h2c) ✅

HTTP/1.1 on the downstream leg (client to Gateway, same plain HTTP listener as Scenario A) and HTTP/2 cleartext (h2c) on the upstream leg (Gateway to backend). This requires `appProtocol: kubernetes.io/h2c` on the Service port and `echo-app` started with h2c support (`ECHO_APP_H2C=true`).

```
curl → Cilium Gateway (HTTP/1.1) → echo-app Service (h2c) → echo-app Pod
```

### HTTPS (TLS Termination)

#### Scenario C — HTTP/1.1 downstream (TLS termination), HTTP/1.1 upstream ✅

The TLS baseline: the Gateway terminates TLS from the client but forwards to the backend over plain HTTP/1.1. No `appProtocol` annotation is needed. This is the simplest HTTPS scenario and a useful reference point before introducing ALPN or h2c. Requires a TLS-terminating `HTTPS` listener on the Gateway.

```
curl → Cilium Gateway (TLS termination) → echo-app Service (HTTP/1.1) → echo-app Pod
```

#### Scenario D — HTTP/2 downstream (ALPN), HTTP/1.1 upstream ✅

Envoy terminates the TLS connection from the client and negotiates HTTP/2 via ALPN, then forwards to the backend over HTTP/1.1. This is the scenario that `gatewayAPI.enableAlpn` is designed for — it requires a TLS-terminating `HTTPS` listener on the Gateway.

```
curl --http2 → Cilium Gateway (TLS, ALPN → HTTP/2) → echo-app Service (HTTP/1.1) → echo-app Pod
```

#### Scenario E — HTTP/2 downstream (ALPN), HTTP/2 upstream (h2c) ✅

Full HTTP/2 on both legs without requiring `BackendTLSPolicy`: Envoy negotiates HTTP/2 with the client via ALPN on the TLS listener, and uses h2c toward the backend via `appProtocol: kubernetes.io/h2c`. Both `gatewayAPI.enableAlpn` and `gatewayAPI.enableAppProtocol` are active. Note that the upstream leg is unencrypted plaintext within the cluster — unlike the not-yet-implemented BackendTLSPolicy scenario where it is TLS-encrypted.

```
curl --http2 → Cilium Gateway (TLS, ALPN → HTTP/2) → echo-app Service (h2c) → echo-app Pod
```

### TLS Passthrough

#### Scenario F — TLS passthrough via TLSRoute ✅

The Gateway passes the raw TLS stream directly to the backend without terminating it. Unlike the other scenarios, Envoy does not inspect or modify the HTTP traffic — the backend is fully responsible for TLS termination and HTTP/2 negotiation (via ALPN). This requires a `TLSRoute` resource (experimental Gateway API before v1.5.0) with a `TLS` listener in `Passthrough` mode, and `echo-app` started with TLS enabled.

```
curl --http2 → Cilium Gateway (TLS passthrough) → echo-app Service (TLS+ALPN, HTTP/2) → echo-app Pod
```

### gRPC

#### Scenario G — gRPC over HTTP/2 cleartext (h2c) ✅

gRPC uses HTTP/2 as its transport. The Gateway receives gRPC traffic on a plain HTTP listener (no TLS) and forwards it to the backend using `appProtocol: kubernetes.io/h2c`. The backend runs `echo-app`'s built-in gRPC server, which responds with request details including the invoked method name.

```
grpcurl → Cilium Gateway (HTTP/2, h2c listener) → echo-app Service (h2c/gRPC) → echo-app Pod
```

#### Scenario H — gRPC over TLS (HTTP/2 via ALPN) ✅

Same as Scenario G but with TLS termination at the Gateway. The client negotiates HTTP/2 via ALPN, allowing encrypted gRPC without requiring the client to know about h2c. The upstream leg still uses `appProtocol: kubernetes.io/h2c` to the same `echo-app-grpc` Service — only the downstream leg changes. Requires `gatewayAPI.enableAlpn: true`.

```
grpcurl → Cilium Gateway (TLS, ALPN → HTTP/2) → echo-app Service (h2c/gRPC) → echo-app Pod
```

### Not Implemented

#### HTTP/2 end-to-end via TLS upstream (BackendTLSPolicy) ❌

Full HTTP/2 on both legs where the Gateway terminates the client connection (ALPN) and re-encrypts toward the backend using TLS. Unlike Scenario F, Envoy is aware of the HTTP traffic on both legs. This requires a `BackendTLSPolicy` resource, which is not yet implemented in Cilium. It will only come in Cilium 1.20+ (tracking issue: [cilium/cilium#31352](https://github.com/cilium/cilium/issues/31352)).

```
curl --http2 → Cilium Gateway (ALPN, HTTP/2) → echo-app Service (TLS/ALPN, HTTP/2) → echo-app Pod
```

This demo currently implements **Scenarios A through H**.

## Prerequisites

- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)
- [cilium CLI](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli) (optional, for status checks)

## Setup

### 1. Add the Cilium Helm Repository

```bash
helm repo add cilium https://helm.cilium.io
helm repo update cilium
```

### 2. Create the KIND Cluster

```bash
kind create cluster --config kind.yaml
```

> **Note:** The `kind.yaml` maps port 8080 (Scenarios A/B), port 8443 (Scenario F), port 8444 (Scenarios C/D/E), and port 50051 (Scenario G) to the host.

### 3. Install the Gateway API CRDs

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.3.0/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml
```

Source: https://docs.cilium.io/en/v1.18/network/servicemesh/gateway-api/gateway-api/#prerequisites

### 4. Install Cilium

**bash/zsh:**

```bash
CLUSTER_NAME=$(kubectl config current-context | sed 's/kind-//')
API_SERVER_HOST="${CLUSTER_NAME}-control-plane"

helm upgrade -i cilium cilium/cilium \
  --wait \
  --version 1.18.8 \
  --namespace kube-system \
  -f helm-values/cilium.yaml \
  --set cluster.name="${CLUSTER_NAME}" \
  --set k8sServiceHost="${API_SERVER_HOST}" \
  --set k8sServicePort=6443
```

**fish:**

```bash
set CLUSTER_NAME (kubectl config current-context | sed 's/kind-//')
set API_SERVER_HOST "$CLUSTER_NAME-control-plane"

helm upgrade -i cilium cilium/cilium \
  --wait \
  --version 1.18.8 \
  --namespace kube-system \
  -f helm-values/cilium.yaml \
  --set cluster.name="$CLUSTER_NAME" \
  --set k8sServiceHost="$API_SERVER_HOST" \
  --set k8sServicePort=6443
```

### 5. Wait for Cilium to Be Ready

```bash
kubectl rollout status -n kube-system deployment/cilium-operator --timeout=180s
kubectl wait --for=condition=Ready nodes --all --timeout=180s
```

### 6. Create the TLS Certificate for the HTTPS Gateway Listener

Scenarios C, D, E, and H use a TLS-terminating Gateway listener that needs a certificate. Generate a self-signed cert and store it as a Secret in `kube-system`:

```bash
openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -keyout tls.key -out tls.crt \
  -subj "/O=Demo Gateway CA" \
  -addext "subjectAltName=DNS:echo-https-plain.127.0.0.1.sslip.io,DNS:echo-https.127.0.0.1.sslip.io,DNS:echo-https-h2c.127.0.0.1.sslip.io,DNS:echo-grpc-tls.127.0.0.1.sslip.io"

kubectl create secret tls echo-https-cert \
  --cert=tls.crt --key=tls.key \
  -n kube-system

rm tls.crt tls.key
```

### 7. Deploy echo-app and the Gateways

A single `echo-app` Deployment serves all scenarios: it listens on port 8080 with h2c (backwards-compatible with HTTP/1.1), port 8443 with TLS, and port 50051 with gRPC. Three Services cover the different upstream protocols: plain HTTP/1.1 (Scenarios A, C, D), h2c (Scenarios B, E), and gRPC/h2c (Scenarios G, H) via the `echo-grpc` Service.

> **Note:** Gateway API CRDs (step 3) must be installed before applying these manifests.

```bash
kubectl apply -f manifests/echo-app.yaml
kubectl apply -f manifests/gateway.yaml
kubectl apply -f manifests/gateway-h2c.yaml
kubectl apply -f manifests/gateway-tls.yaml
kubectl apply -f manifests/gateway-grpc.yaml
```

### 8. Wait for the Deployment

```bash
kubectl rollout status -n echo-app deployment/echo-app --timeout=120s
kubectl wait -n echo-app --for=condition=Ready pod -l app=echo-app --timeout=60s
```

## Testing

### HTTP

#### Scenario A — HTTP/1.1 downstream, HTTP/1.1 upstream

Send a request to the Gateway. Envoy forwards it to the backend over HTTP/1.1. The `http_version` field in the response reflects what the pod received — HTTP/1.1 — and the Envoy-injected headers (`X-Envoy-Internal`, `X-Request-Id`) confirm the request was proxied through the Gateway:

```bash
curl -s http://echo.127.0.0.1.sslip.io:8080/ | jq
```

**Expected output:**

```json
{
  "timestamp": "2026-04-08T12:34:05Z",
  "message": "http2-demo",
  "hostname": "echo-app-5c4ccf4565-m56gf",
  "listener": "HTTP",
  "node": "echo-http2-control-plane",
  "source_ip": "10.244.0.216",
  "http_version": "HTTP/1.1",
  "http_method": "GET",
  "http_endpoint": "/",
  "headers": {
    "Accept": [
      "*/*"
    ],
    "User-Agent": [
      "curl/8.7.1"
    ],
    "X-Envoy-Internal": [
      "true"
    ],
    "X-Forwarded-For": [
      "172.18.0.1"
    ],
    "X-Forwarded-Proto": [
      "http"
    ],
    "X-Request-Id": [
      "ebdb93ee-324a-4da2-b41d-a93dc2b94c2e"
    ]
  }
}
```

Key fields in the response:

- **`listener`**: `HTTP` — the pod received a plain HTTP request (no TLS, no h2c).
- **`http_version`**: `HTTP/1.1` — both legs use HTTP/1.1.
- **`X-Envoy-Internal`**: injected by Envoy, confirming the request was proxied through the Gateway.
- **`X-Forwarded-For`**: the original client IP, added by Envoy.
- **`X-Forwarded-Proto`**: `http` — Envoy records the downstream protocol.
- **`X-Request-Id`**: injected by Envoy for request tracing.

#### Run Multiple Requests to Test Load Balancing

With `replicas: 2`, requests are distributed across both pods. The `hostname` field shows which pod served the request:

**bash/zsh:**

```bash
for i in $(seq 1 6); do
  curl -s http://echo.127.0.0.1.sslip.io:8080/ | jq -r '.hostname + " (" + .http_version + ")"'
done
```

**fish:**

```bash
for i in (seq 1 6)
  curl -s http://echo.127.0.0.1.sslip.io:8080/ | jq -r '.hostname + " (" + .http_version + ")"'
end
```

#### Scenario B — HTTP/1.1 downstream, HTTP/2 upstream (h2c)

Send a request to the Gateway. Envoy receives it over HTTP/1.1 (same plain HTTP listener as Scenario A) and forwards it to the backend over HTTP/2 cleartext (h2c) because of `appProtocol: kubernetes.io/h2c` on the Service. The `http_version` field in the response shows HTTP/2.0, confirming the upstream leg is h2c:

```bash
curl -s http://echo-h2c.127.0.0.1.sslip.io:8080/ | jq
```

**Expected output:**

```json
{
  "timestamp": "2026-04-08T12:34:35Z",
  "message": "http2-demo",
  "hostname": "echo-app-5c4ccf4565-cq5pj",
  "listener": "H2C",
  "node": "echo-http2-control-plane",
  "source_ip": "10.244.0.216",
  "http_version": "HTTP/2.0",
  "http_method": "GET",
  "http_endpoint": "/",
  "headers": {
    "Accept": [
      "*/*"
    ],
    "User-Agent": [
      "curl/8.7.1"
    ],
    "X-Envoy-Internal": [
      "true"
    ],
    "X-Forwarded-For": [
      "172.18.0.1"
    ],
    "X-Forwarded-Proto": [
      "http"
    ],
    "X-Request-Id": [
      "b7a4b235-9368-4848-81a4-5075e44036d3"
    ]
  }
}
```

The key difference from Scenario A is `"listener": "H2C"` and `"http_version": "HTTP/2.0"` — the backend pod received the request as HTTP/2, proving the upstream leg uses h2c.

##### Verify the Downstream Is HTTP/1.1

The `http_version` field confirms the upstream leg is HTTP/2. To confirm the downstream leg (curl → Envoy) is HTTP/1.1, check the response protocol in verbose output — it will show `< HTTP/1.1`:

```bash
curl -sv http://echo-h2c.127.0.0.1.sslip.io:8080/ 2>&1 | grep -E '^[<>] (HTTP|GET)'
```

Expected:

```
> GET / HTTP/1.1
< HTTP/1.1 200 OK
```

### HTTPS (TLS Termination)

#### Scenario C — HTTP/1.1 downstream (TLS termination), HTTP/1.1 upstream

Send a request over HTTPS to the Gateway. Envoy terminates TLS and forwards to the backend over plain HTTP/1.1. Use `--http1.1` to explicitly prevent curl from offering h2 via ALPN — without it, curl negotiates HTTP/2 by default over TLS, which would make this scenario identical to Scenario D:

```bash
curl -sk --http1.1 https://echo-https-plain.127.0.0.1.sslip.io:8444/ | jq
```

**Expected output:**

```json
{
  "timestamp": "2026-04-09T10:51:29Z",
  "message": "http2-demo",
  "hostname": "echo-app-665567bb7-msrlv",
  "listener": "HTTP",
  "node": "echo-http2-control-plane",
  "source_ip": "10.244.0.117",
  "http_version": "HTTP/1.1",
  "http_method": "GET",
  "http_endpoint": "/",
  "headers": {
    "Accept": [
      "*/*"
    ],
    "User-Agent": [
      "curl/8.7.1"
    ],
    "X-Envoy-Internal": [
      "true"
    ],
    "X-Forwarded-For": [
      "172.18.0.1"
    ],
    "X-Forwarded-Proto": [
      "https"
    ],
    "X-Request-Id": [
      "0c0cd759-283d-49fc-8425-1a445e721e07"
    ]
  }
}
```

The `"listener": "HTTP"` and `"http_version": "HTTP/1.1"` confirm that the backend received a plain HTTP/1.1 request. The Gateway uses the manually provisioned certificate to terminate TLS; the backend is unaware of TLS. Note `"X-Forwarded-Proto": "https"` compared to Scenario A's `"http"`, confirming the downstream leg was HTTPS.

##### Verify Both Legs

Confirm HTTP/1.1 on both the downstream and upstream legs:

```bash
curl -skv --http1.1 https://echo-https-plain.127.0.0.1.sslip.io:8444/ 2>&1 | grep -E '^[<>] (HTTP|GET)'
```

Expected:

```
> GET / HTTP/1.1
< HTTP/1.1 200 OK
```

#### Scenario D — HTTP/2 downstream (ALPN), HTTP/1.1 upstream

With a TLS listener, Envoy negotiates HTTP/2 with the client via ALPN. The `--http2` flag is used here to explicitly request HTTP/2 — though curl would also negotiate it by default over TLS. The backend still receives HTTP/1.1:

```bash
curl -sk --http2 https://echo-https.127.0.0.1.sslip.io:8444/ | jq
```

**Expected output:**

```json
{
  "timestamp": "2026-04-08T12:35:38Z",
  "message": "http2-demo",
  "hostname": "echo-app-5c4ccf4565-m56gf",
  "listener": "HTTP",
  "node": "echo-http2-control-plane",
  "source_ip": "10.244.0.216",
  "http_version": "HTTP/1.1",
  "http_method": "GET",
  "http_endpoint": "/",
  "headers": {
    "Accept": [
      "*/*"
    ],
    "User-Agent": [
      "curl/8.7.1"
    ],
    "X-Envoy-Internal": [
      "true"
    ],
    "X-Forwarded-For": [
      "172.18.0.1"
    ],
    "X-Forwarded-Proto": [
      "https"
    ],
    "X-Request-Id": [
      "ee985981-5a55-45c0-a3b7-5bf1460df1bf"
    ]
  }
}
```

The `"http_version": "HTTP/1.1"` confirms the upstream leg is still HTTP/1.1. To confirm HTTP/2 on the downstream leg:

```bash
curl -skv --http2 https://echo-https.127.0.0.1.sslip.io:8444/ 2>&1 | grep -E 'ALPN|^[<>] (HTTP|GET)'
```

Expected:

```
* ALPN: curl offers h2,http/1.1
* ALPN: server accepted h2
> GET / HTTP/2
< HTTP/2 200
```

#### Scenario E — HTTP/2 downstream (ALPN), HTTP/2 upstream (h2c)

Full HTTP/2 on both legs: Envoy negotiates HTTP/2 with the client via ALPN and uses h2c toward the backend. The `--http2` flag is used for clarity:

```bash
curl -sk --http2 https://echo-https-h2c.127.0.0.1.sslip.io:8444/ | jq
```

**Expected output:**

```json
{
  "timestamp": "2026-04-08T12:36:09Z",
  "message": "http2-demo",
  "hostname": "echo-app-5c4ccf4565-m56gf",
  "listener": "H2C",
  "node": "echo-http2-control-plane",
  "source_ip": "10.244.0.216",
  "http_version": "HTTP/2.0",
  "http_method": "GET",
  "http_endpoint": "/",
  "headers": {
    "Accept": [
      "*/*"
    ],
    "User-Agent": [
      "curl/8.7.1"
    ],
    "X-Envoy-Internal": [
      "true"
    ],
    "X-Forwarded-For": [
      "172.18.0.1"
    ],
    "X-Forwarded-Proto": [
      "https"
    ],
    "X-Request-Id": [
      "a04d189e-469e-40ee-941c-4ff59c91d97f"
    ]
  }
}
```

Both `"listener": "H2C"` and `"http_version": "HTTP/2.0"` confirm end-to-end HTTP/2. To verify both legs explicitly:

```bash
curl -skv --http2 https://echo-https-h2c.127.0.0.1.sslip.io:8444/ 2>&1 | grep -E 'ALPN|^[<>] (HTTP|GET)'
```

Expected:

```
* ALPN: curl offers h2,http/1.1
* ALPN: server accepted h2
> GET / HTTP/2
< HTTP/2 200
```

### TLS Passthrough

#### Scenario F — TLS passthrough via TLSRoute

The Gateway passes the raw TLS stream to the backend without terminating it. `echo-app` handles TLS itself using a self-signed certificate and negotiates HTTP/2 via ALPN. Use `-k` to skip certificate verification and `--http2` to request HTTP/2:

```bash
curl -sk --http2 https://echo-tls.127.0.0.1.sslip.io:8443/ | jq
```

**Expected output:**

```json
{
  "timestamp": "2026-04-08T12:35:08Z",
  "message": "http2-demo",
  "hostname": "echo-app-5c4ccf4565-cq5pj",
  "listener": "TLS",
  "node": "echo-http2-control-plane",
  "source_ip": "10.244.0.216",
  "http_version": "HTTP/2.0",
  "http_method": "GET",
  "http_endpoint": "/",
  "headers": {
    "Accept": [
      "*/*"
    ],
    "User-Agent": [
      "curl/8.7.1"
    ]
  }
}
```

The `"listener": "TLS"` and `"http_version": "HTTP/2.0"` confirm that the backend pod received and terminated the TLS connection directly, with HTTP/2 negotiated via ALPN — the Gateway was fully transparent. Also note the absence of Envoy-injected headers (`X-Envoy-Internal`, `X-Forwarded-For`, `X-Request-Id`): since Envoy does not terminate TLS, it cannot inspect or modify the encrypted HTTP traffic and therefore adds no headers.

To confirm the TLS passthrough on the client side, inspect the certificate. The cert is self-signed by `echo-app` itself — the Gateway is fully transparent and presents no certificate of its own. Use `-k` to skip verification so the handshake completes and the cert details are printed:

```bash
curl -skv --http2 https://echo-tls.127.0.0.1.sslip.io:8443/ 2>&1 | grep -E 'issuer|subject|SSL certificate| HTTP'
```

Expected output will show a self-signed certificate (issuer == subject) and HTTP/2 negotiated via ALPN:

```
*  subject: O=Echo Inc.
*  issuer: O=Echo Inc.
*  SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway.
* using HTTP/2
> GET / HTTP/2
< HTTP/2 200
```

### gRPC

#### Scenario G — gRPC over HTTP/2 cleartext (h2c)

[`grpcurl`](https://github.com/fullstorydev/grpcurl) is used to send gRPC requests. Install it with `brew install grpcurl` (macOS) or follow the [installation guide](https://github.com/fullstorydev/grpcurl#installation).

The `echo-app` exposes a gRPC service (`echo.EchoService/Echo`). Use `grpcurl` with the `-plaintext` flag since the Gateway listener is plain HTTP/2 (no TLS):

```bash
grpcurl -plaintext \
  echo-grpc.127.0.0.1.sslip.io:50051 \
  echo.EchoService/Echo
```

**Expected output:**

```json
{
  "timestamp": "2026-04-09T10:58:12Z",
  "message": "http2-demo",
  "hostname": "echo-app-665567bb7-msrlv",
  "listener": "gRPC",
  "node": "echo-http2-control-plane",
  "sourceIp": "10.244.0.117",
  "grpcMethod": "/echo.EchoService/Echo"
}
```

The response confirms the gRPC call reached the `echo-app` backend through the Cilium Gateway. The Gateway listener is a plain HTTP listener on port 50051; Cilium forwards the HTTP/2 frames to the backend using `appProtocol: kubernetes.io/h2c` on the `echo-grpc` Service.

#### Scenario H — gRPC over TLS (HTTP/2 via ALPN)

Same as Scenario G but over TLS. The Gateway terminates TLS on the existing HTTPS listener (port 8444) and negotiates HTTP/2 via ALPN. Use `-insecure` to skip verification of the self-signed certificate:

```bash
grpcurl -insecure \
  echo-grpc-tls.127.0.0.1.sslip.io:8444 \
  echo.EchoService/Echo
```

**Expected output:**

```json
{
  "timestamp": "2026-04-09T10:59:47Z",
  "message": "http2-demo",
  "hostname": "echo-app-665567bb7-msrlv",
  "listener": "gRPC",
  "node": "echo-http2-control-plane",
  "sourceIp": "10.244.0.117",
  "grpcMethod": "/echo.EchoService/Echo"
}
```

The response is identical to Scenario G — the difference is entirely on the transport layer: the client connected over TLS and negotiated HTTP/2 via ALPN rather than using h2c directly. The upstream leg (Gateway → backend) remains h2c via `appProtocol: kubernetes.io/h2c`.

To confirm TLS termination and ALPN negotiation on the downstream leg, inspect the TLS handshake directly:

```bash
echo "" | openssl s_client -connect echo-grpc-tls.127.0.0.1.sslip.io:8444 -alpn h2 2>&1 | grep -E 'Protocol|ALPN|issuer|subject|Cipher'
```

Expected:

```
subject=/O=Demo Gateway CA
issuer=/O=Demo Gateway CA
New, TLSv1/SSLv3, Cipher is AEAD-CHACHA20-POLY1305-SHA256
ALPN protocol: h2
    Protocol  : TLSv1.3
    Cipher    : AEAD-CHACHA20-POLY1305-SHA256
```

`ALPN protocol: h2` confirms HTTP/2 was negotiated on the downstream leg. The `issuer=/O=Demo Gateway CA` identifies the manually provisioned certificate (same cert used by Scenarios C, D, and E) — in contrast to Scenario F, where the TLS passthrough exposes `echo-app`'s own self-signed certificate with `issuer=/O=Echo Inc.`.

## Teardown

```bash
kind delete cluster --name echo-http2
```

## Repository Structure

```
cilium-gwapi-http2-demo/
├── README.md                   # This file
├── kind.yaml                   # KIND cluster configuration
├── helm-values/
│   └── cilium.yaml             # Cilium Helm values (gatewayAPI.enableAlpn, gatewayAPI.enableAppProtocol)
└── manifests/
    ├── echo-app.yaml           # All scenarios: one Deployment + Services (plain, h2c, grpc)
    ├── gateway.yaml            # Scenarios A/C/D/E: Cilium Gateway (HTTP + HTTPS listeners) and HTTPRoutes
    ├── gateway-h2c.yaml        # Scenario B: HTTPRoute for h2c Service
    ├── gateway-tls.yaml        # Scenario F: TLS Gateway listener and TLSRoute
    └── gateway-grpc.yaml       # Scenarios G/H: gRPC Gateway listener and GRPCRoutes (h2c and TLS)
```

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](./LICENSE) file for details.
