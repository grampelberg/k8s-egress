# Overview

This repo contains a proof-of-concept to enable Linkerd's metrics, retries, and other features for external (off-cluster or 3rd-party) HTTP calls. For example, with this POC, all calls to `api.github.com` can be monitored for success rate, retried automatically, and so on.

# Background

Calls to third-party APIs are typically TLS'd by the application directly (e.g. the client connects to `httpS://api.github.com`). However, Linkerd can only provide metrics and reliability features for *non-TLS'd* HTTP connections (e.g. `http://api.github.com`)--which it will then add TLS to, on behalf of the application. Thus, if we want metrics, we must provide non-TLS'd connections to Linkerd.

However, simply changing all third-party HTTP calls from HTTPS to plaintext HTTP is risky: if these connections are ever accidentally made in a non-Linkerd-enabled context they may traverse the open internet in plaintext, potentially exposing sensitive data.

Thus, the purpose of this POC is to provide a *safe* mechanism by which your application can initiate a plaintext HTTP call to a third party API (to be TLS'd by Linkerd), by ensuring that these calls will fail unless they are run in the production context.

## How it works

This POC installs an egress proxy and DNS rules such that all calls to `FOO.egress.local` resolve to this proxy, which in turn encrypts the connection and proxies it to `FOO`. For example, a call to `api.github.com.egress.local` will be proxied to `api.github.com`. Linkerd will mTLS the connection from the application to the egress proxy; the egress proxy will TLS the connection to the third party.

Then, in application code, all calls in the production context *only* to `httpS://FOO` are replacted with calls to `http://FOO.egress.local`.

Critically, this approach "fails safely": if the application is not running in the production context (with Linkerd and the egress proxy), the `.egress.local` domain will not resolve and this unencrypted call will fail. Additionally, this provides a convenient point for enforcing egress control.

## Installation

```bash
kubectl apply -f egress.yml
kubectl apply -f kube-dns.yml
```

Note: If you're already using coredns, just add the following to your configmap:

```
rewrite name regex (.*)\.egress.local proxy.egress.svc.cluster.local
```

## Usage

In your application code, when running in the production context *only*, replace calls to `httpS://FOO` with calls to `http://FOO.egress.local`. For example, calling of addressing `httpS://api.github.com`, call `http://api.github.com.egress.local`.

Metrics for calls to `FOO.egress.local` will then be available alongside all other Linkerd metrics.

# Future work

- Restricting outbound traffic to specific egress pods.
- Inspecting HTTP metrics at a central location.
- Applying egress policy.
