# ExternalDNS demo — Kubernetes manifests

Automates a Cloudflare DNS record straight from a Kubernetes Ingress.

- **Domain / zone:** `devopsdetours-lab.com` (must be on Cloudflare)
- **Hostname created:** `shop.devopsdetours-lab.com`
- **Demo app:** [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo)
  (microservices-demo) — used across the whole series
- **Ingress controller:** Traefik
- **DNS provider:** Cloudflare (ExternalDNS `--source=ingress`)

```
Browser ─DNS─▶ Cloudflare ──▶ AWS ELB ──▶ Traefik ──▶ Service: frontend ──▶ pods
                  ▲
                  │ CNAME + TXT (UPSERT)
            ExternalDNS  ◀── watches the Ingress (host + LB status)
```

## Files
| File | What it is |
|------|------------|
| `external-dns-values.yaml` | Helm values for the ExternalDNS chart (Cloudflare provider, Ingress source) |
| `namespace.yaml` | The `shop` namespace for the app |
| `ingress.yaml` | Traefik Ingress for `shop.devopsdetours-lab.com` → `frontend:80` |
| `../.env.example` | Template for the Cloudflare token env var (copy → `.env`, gitignored) |

## Prerequisites
- EKS cluster from [`../terraform`](../terraform) and a working `kubectl` context
- `helm`
- `devopsdetours-lab.com` active on Cloudflare (nameservers pointed at Cloudflare)
- A Cloudflare API token (scopes in [`../.env.example`](../.env.example))

---

## Step 1 — Install Traefik

> A Helm **chart** is a pre-packaged Kubernetes app — all the YAML (Deployment,
> Service, RBAC…) bundled so you install it with one command. Charts come from a
> **repo** (a catalog you add once). Below: `traefik-charts` is the repo alias,
> `traefik-ingress` is our install's name (the "release"), and `traefik` is the
> chart itself.

```bash
helm repo add traefik-charts https://traefik.github.io/charts
helm repo update
helm install traefik-ingress traefik-charts/traefik \
  -n traefik --create-namespace \
  --set providers.kubernetesIngress.publishedService.enabled=true
```
`publishedService.enabled=true` makes Traefik write its **ELB hostname into every
Ingress's `status.loadBalancer`** — that's the target ExternalDNS reads. Without it,
the Ingress has no address and ExternalDNS has nothing to point the CNAME at.

On EKS this creates a `LoadBalancer` Service → an AWS ELB. Confirm it gets an address
(the Service is named after the release):
```bash
kubectl get svc -n traefik traefik-ingress -w   # wait for EXTERNAL-IP (an *.elb.amazonaws.com host)
```

## Step 2 — Deploy Online Boutique (the app)
```bash
kubectl apply -f namespace.yaml
kubectl apply -n shop \
  -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/release/kubernetes-manifests.yaml

# we expose the app through the Ingress, so the bundled public LoadBalancer
# Service isn't needed (saves an extra ELB):
kubectl delete svc frontend-external -n shop --ignore-not-found

kubectl get pods -n shop -w   # wait until everything is Running
```
The upstream manifest creates a `frontend` Service (`ClusterIP`, port **80** →
targetPort 8080) — which is exactly what `ingress.yaml` points at, so nothing in
the app needs editing.

> Resourcing: Online Boutique is ~11 services. The 2× `t3.medium` spot nodes from
> the Terraform should hold it; if pods stay `Pending`, bump `desired_nodes` /
> `max_nodes` in [`../terraform/variables.tf`](../terraform/variables.tf).

## Step 3 — Create the Cloudflare token Secret (from env, not a file)
```bash
cp ../.env.example ../.env        # then edit ../.env and paste your token (gitignored)
source ../.env                    # exports CF_API_TOKEN

kubectl create namespace external-dns
kubectl create secret generic cloudflare-token -n external-dns \
  --from-literal=cloudflare_api_token="$CF_API_TOKEN"
```
The token only ever lives in your shell env — it never gets written into a
committed (or on-disk) manifest.

## Step 4 — Deploy ExternalDNS (Helm)
```bash
# external-dns-controller = release · external-dns-charts = repo alias · external-dns = chart
helm repo add external-dns-charts https://kubernetes-sigs.github.io/external-dns/
helm repo update
helm install external-dns-controller external-dns-charts/external-dns \
  -n external-dns -f external-dns-values.yaml

kubectl logs -n external-dns deploy/external-dns-controller -f
```
The chart creates the ServiceAccount, RBAC and Deployment. All the meaningful
config (provider, source, domain-filter, policy, txt-owner-id, txt-prefix) lives
in [`external-dns-values.yaml`](external-dns-values.yaml).

## Step 5 — Create the Ingress
```bash
kubectl apply -f ingress.yaml
kubectl get ingress -n shop -w   # wait for ADDRESS (Traefik's ELB host) to appear
```

## Step 6 — Verify the handoff
```bash
# Ingress has an address (Traefik published it):
kubectl get ingress -n shop

# ExternalDNS acted on it:
kubectl logs -n external-dns deploy/external-dns-controller | grep -i shop
#   ... CREATE: shop.devopsdetours-lab.com  CNAME -> k8s-...elb.amazonaws.com
#   ... CREATE: extdns-shop.devopsdetours-lab.com  TXT (ownership)

# DNS resolves and the shop answers:
dig +short shop.devopsdetours-lab.com
curl -s http://shop.devopsdetours-lab.com | head
```
In the Cloudflare dashboard you should see a **CNAME** `shop` and a **TXT**
`extdns-shop` (the ownership record from Scene 4).

## Cleanup
```bash
kubectl delete -f ingress.yaml          # policy=sync → ExternalDNS deletes the records
kubectl delete -n shop -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/release/kubernetes-manifests.yaml
helm uninstall external-dns-controller -n external-dns
kubectl delete namespace external-dns   # removes the cloudflare-token Secret too
kubectl delete -f namespace.yaml
helm uninstall traefik-ingress -n traefik
```

## Notes & gotchas
- **`policy: sync`** (in `external-dns-values.yaml`) lets ExternalDNS delete records when
  the Ingress goes away — great for the demo. Use `upsert-only` in production if you
  never want automatic deletes.
- **`--txt-prefix=extdns-`** keeps the ownership TXT from colliding with the CNAME
  (you can't have a CNAME and a TXT at the same name).
- **Cloudflare proxy:** the Ingress sets
  `external-dns.alpha.kubernetes.io/cloudflare-proxied: "false"` so the record is
  DNS-only and `dig`/`curl` hit the ELB directly. Flip to `"true"` for the orange
  cloud. (HTTPS is the cert-manager video.)
- **Propagation:** first resolution can lag a minute or two. ExternalDNS reconciles
  on its `--interval` (default 1m); uncomment `--interval=30s` for a snappier demo.

## Architecture

```text
  File labels are relative to k8s/ unless prefixed with ../.

  Live traffic (Cloudflare record is DNS-only)

  +-------------------------------------+
  | User -> Browser/Curl                |
  +------------------+------------------+
                     |
                     | DNS: shop.devopsdetours-lab.com
                     v
  +-------------------------------------+
  | Cloudflare DNS zone                 |
  | devopsdetours-lab.com               |
  | CNAME shop -> <traefik-elb>         |
  | TXT extdns-shop -> owner data       |
  | managed by external-dns-values.yaml |
  | hostname from ingress.yaml          |
  +------------------+------------------+
                     |
                     | HTTP to <traefik-elb>
                     v
  +-------------------------------------+
  | AWS ELB                             |
  | Service/traefik-ingress             |
  | created by README.md Step 1         |
  +------------------+------------------+
                     |
                     v
  ========== Amazon EKS cluster (../terraform/) ==========
                     |
  namespace: traefik
  +------------------v------------------+
  | Traefik controller                  |
  | ingressClassName=traefik            |
  | installed by README.md Step 1       |
  +------------------+------------------+
                     |
                     | Host: shop.devopsdetours-lab.com
                     v
  namespace: shop (namespace.yaml)
  +-------------------------------------+
  | Ingress/frontend                    |
  | file: ingress.yaml                  |
  | backend: frontend:80                |
  +------------------+------------------+
                     |
                     v
  +-------------------------------------+
  | Service/frontend                    |
  | ClusterIP :80 -> :8080              |
  | upstream: kubernetes-manifests.yaml |
  | applied in README.md Step 2         |
  +------------------+------------------+
                     |
                     v
  +-------------------------------------+
  | Pod/frontend                        |
  | Online Boutique UI                  |
  | upstream: kubernetes-manifests.yaml |
  | applied in README.md Step 2         |
  +------------------+------------------+
                     |
                     v
  +-------------------------------------------------------------+
  | Online Boutique backend services                            |
  | upstream: kubernetes-manifests.yaml                         |
  | applied in README.md Step 2                                 |
  | cartservice, productcatalogservice, currencyservice,        |
  | checkoutservice, shippingservice, paymentservice,           |
  | emailservice, recommendationservice, adservice,             |
  | redis-cart, loadgenerator                                   |
  +-------------------------------------------------------------+


  Background reconcile (ExternalDNS is outside the live request path)

  namespace: external-dns
  +-------------------------------------+       watches       +-------------------------------------+
  | ExternalDNS controller              | ------------------> | Ingress/frontend (shop)             |
  | values: external-dns-values.yaml    | <------------------ | file: ingress.yaml                  |
  | source=ingress, policy=sync         |   host + LB status  | host + status.loadBalancer          |
  +------------------+------------------+                     +-------------------------------------+
                     |
                     | reads CF_API_TOKEN
                     v
  +-------------------------------------+
  | Secret/cloudflare-token             |
  | created by README.md Step 3         |
  | token source: ../.env               |
  | template: ../.env.example           |
  +------------------+------------------+
                     |
                     | Cloudflare API sync:
                     | CNAME shop + TXT extdns-shop
                     | files: external-dns-values.yaml,
                     |        ingress.yaml
                     v
  +-------------------------------------+
  | Cloudflare DNS zone                 |
  | devopsdetours-lab.com               |
  | file: external-dns-values.yaml      |
  +-------------------------------------+

  If ExternalDNS stops, existing traffic keeps flowing. Only new or changed DNS
  records pause until it comes back.
```
