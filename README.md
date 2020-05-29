# Kubernetes Egress

- Restrict outbound traffic to specific egress pods.
- Inspect HTTP metrics at a central location.
- Apply service mesh policy.

## Install

```bash
kubectl apply -f egress.yml
kubectl apply -f kube-dns.yml
```

Note: If you're already using coredns, just add the following to your configmap:

```
rewrite name regex (.*)\.egress.local proxy.egress.svc.cluster.local
```

## Usage

In your applications, instead of addressing `https://api.github.com`, use:

```
http://api.github.com.egress.local
```
