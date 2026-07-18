# Kubernetes manifests

The manifests ExternalDNS and Traefik use in this demo. For the full step-by-step
deploy, see **[../walkthrough.md](../walkthrough.md)**.

| File | What it is |
|------|------------|
| [`namespace.yaml`](namespace.yaml) | The `shop` namespace for the app |
| [`ingress.yaml`](ingress.yaml) | The Traefik Ingress — declares `shop.<your-domain>`; the single source of truth ExternalDNS reads |
| [`external-dns-values.yaml`](external-dns-values.yaml) | Helm values for ExternalDNS (Cloudflare provider, `source=ingress`, sync policy, TXT ownership) |

> The Cloudflare API token is **not** here — it's created from a local `.env`
> (see [`../.env.example`](../.env.example)) so it never lands in a committed file.
