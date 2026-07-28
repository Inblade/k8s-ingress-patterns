# k8s-ingress-patterns

Battle-tested ingress-nginx + cert-manager patterns: TLS automation,
canary releases, rate limiting, auth gating, and wildcard certificates.
A reference cookbook distilled from running these controllers in
production — copy the pattern you need and adjust hosts/services.

## Contents

```
.
├── manifests/
│   └── clusterissuer-letsencrypt.yaml  # LE staging + prod ClusterIssuers (HTTP-01)
├── patterns/
│   ├── basic-tls.yaml                  # HTTPS with auto-issued/renewed cert
│   ├── canary.yaml                     # Weighted traffic split via canary annotations
│   ├── rate-limit.yaml                 # Per-client rps/connection limits at the edge
│   ├── basic-auth.yaml                 # htpasswd gate for staging/internal tools
│   └── wildcard-dns01.yaml             # *.example.com via DNS-01 (Cloudflare example)
├── LICENSE
└── README.md
```

## When to use what

| Pattern | Use when | Avoid when |
|---|---|---|
| `basic-tls` | Any public HTTPS service — this is the default starting point | Internal-only services that never leave the cluster |
| `canary` | Gradual rollout of a risky release; need % traffic split without a service mesh | You need metric-driven automated promotion — use Argo Rollouts/Flagger (they drive these same annotations) |
| `rate-limit` | Public APIs, login/search endpoints, scraping abuse | Billing-grade per-token quotas (use an API gateway); multi-replica controllers if you need exact global limits |
| `basic-auth` | Staging environments, dashboards without built-in auth | Anything customer-facing — graduate to oauth2-proxy/SSO |
| `wildcard-dns01` | Many short-lived subdomains (preview envs, per-tenant hosts); LE rate-limit pressure | A handful of stable hosts — per-host HTTP-01 certs are simpler and don't need DNS API credentials |

## Usage

```bash
# Prerequisites: ingress-nginx + cert-manager installed, e.g.
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx -n ingress-nginx --create-namespace
helm upgrade --install cert-manager cert-manager \
  --repo https://charts.jetstack.io -n cert-manager --create-namespace \
  --set crds.enabled=true

# 1. Issuers (edit the email first)
kubectl apply -f manifests/clusterissuer-letsencrypt.yaml

# 2. Pick a pattern, replace example.com hosts and Service names, apply
kubectl apply -f patterns/basic-tls.yaml

# Watch issuance
kubectl -n app get certificate,certificaterequest,order,challenge
```

## Operating notes

- **Always test issuance against `letsencrypt-staging` first.** Production
  Let's Encrypt rate limits are per registered domain per week; a broken
  rollout can lock you out for days. Swap the annotation to
  `letsencrypt-prod` only after the staging cert issues cleanly.
- **Rate limits are per controller replica** — with 3 nginx pods the
  effective limit is up to 3x the annotation. Details in `patterns/rate-limit.yaml`.
- **Real client IPs need controller-level config** (`externalTrafficPolicy:
  Local` or `use-forwarded-headers`) or rate limiting and IP whitelisting
  will apply to your load balancer's address instead.
- **Wildcards require DNS-01** and an explicit Certificate resource;
  Ingress-annotation-driven issuance covers only per-host certs.

## Compatibility

Manifests use `networking.k8s.io/v1` Ingress (K8s 1.19+),
`cert-manager.io/v1` (cert-manager 1.x), and annotations current for
ingress-nginx 1.x.

## License

MIT — see [LICENSE](LICENSE).
