# ExternalDNS on EKS with Cloudflare — hands-on walkthrough

Companion guide to the video. Follow along and you'll go from an empty cluster to a
real hostname — `shop.devopsdetours-lab.com` — resolving to your app **automatically**,
with no DNS records touched by hand.

**What you'll build**

- An **Amazon EKS** cluster (via Terraform)
- **Traefik** as the ingress controller (gets an AWS load balancer)
- **Online Boutique** as the demo app
- **ExternalDNS**, which watches your Ingress and keeps **Cloudflare** in sync

The whole idea: you declare the hostname once, in an Ingress. ExternalDNS turns it
into a Cloudflare `CNAME` (plus a `TXT` ownership record) and keeps it in sync —
completely out of your users' traffic path.

Commands are run from the repo root unless noted.

---

## Before you start

**Tools**

```bash
terraform version              # >= 1.5.7
kubectl version --client
helm version
aws sts get-caller-identity    # your AWS credentials work
```

**Accounts**

- An AWS account (the Terraform config creates a small EKS cluster — this costs money while it runs).
- A domain managed by **Cloudflare** — the zone must be active (nameservers pointed at Cloudflare).
- A **Cloudflare API token** scoped to that zone: **Zone › DNS › Edit** + **Zone › Zone › Read**.

**⚠️ Values to change to your own**

This guide uses my demo values. Swap these for yours before you run anything —
everything else works unchanged.

| What | This guide uses | Change it in |
|------|-----------------|--------------|
| Cloudflare zone | `devopsdetours-lab.com` | [k8s/external-dns-values.yaml](k8s/external-dns-values.yaml) → `domainFilters` |
| Hostname | `shop.devopsdetours-lab.com` | [k8s/ingress.yaml](k8s/ingress.yaml) → `rules[].host` |
| Owner ID (TXT registry) | `devopsdetours` | [k8s/external-dns-values.yaml](k8s/external-dns-values.yaml) → `txtOwnerId` |
| Cluster name / region | `externaldns-demo` / `eu-central-1` | [terraform/](terraform/) + the connect step below |

---

## 1. Provision the cluster (Terraform)

```bash
cd terraform
terraform init
terraform apply        # ~15 min for EKS
cd ..
```

✅ You should see `Apply complete!` and some outputs (cluster name, a `configure_kubectl` command, etc.).

## 2. Connect kubectl

```bash
# writes kubeconfig for the new cluster and switches your context to it:
aws eks update-kubeconfig --region eu-central-1 --name externaldns-demo
kubectl get nodes      # nodes should be Ready
```

## 3. Install Traefik (ingress controller)

A Helm **chart** is a pre-packaged Kubernetes app (all the YAML bundled together);
charts live in a **repo** you add once; installing one creates a **release**.

```bash
helm repo add traefik-charts https://traefik.github.io/charts
helm repo update

# traefik-ingress  = release name (your choice)
# traefik-charts/  = repo alias
# /traefik         = chart name
helm install traefik-ingress traefik-charts/traefik \
  -n traefik --create-namespace

# wait for the AWS load balancer to be assigned:
kubectl get svc -n traefik traefik-ingress -w
```

✅ The `traefik-ingress` Service gets an `EXTERNAL-IP` like `...elb.amazonaws.com`.
That hostname is what ExternalDNS will eventually point your CNAME at.

> **We installed *without* one important flag on purpose.** In [step 7](#7-apply-the-ingress--and-see-why-the-flag-matters)
> we'll add `publishedService` and watch exactly what it fixes — the clearest way
> to understand what it does. **Just want it working?** Add
> `--set providers.kubernetesIngress.publishedService.enabled=true` to the install
> above and skip the "before" part of step 7.

## 4. Deploy Online Boutique (the app)

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -n shop \
  -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/release/kubernetes-manifests.yaml

# the app ships its own public LoadBalancer — we don't want that; we expose it via
# Traefik instead, so remove it:
kubectl delete svc frontend-external -n shop --ignore-not-found

kubectl get pods -n shop -w     # wait until all pods are Running
```

✅ Every pod `Running`; `kubectl get svc -n shop frontend` shows a ClusterIP on port 80.

## 5. Cloudflare token → Kubernetes Secret

Load the token from your shell environment so it never gets written into a file.

```bash
cp .env.example .env
# edit .env → paste your Cloudflare API token  (.env is gitignored)
source .env                              # exports CF_API_TOKEN

kubectl create namespace external-dns
kubectl create secret generic cloudflare-token -n external-dns \
  --from-literal=cloudflare_api_token="$CF_API_TOKEN"
kubectl get secret -n external-dns cloudflare-token   # exists
```

## 6. Install ExternalDNS

Its config lives in [k8s/external-dns-values.yaml](k8s/external-dns-values.yaml). The key lines:

- `sources: ingress` — watch Ingress objects (this is why declaring the hostname in an Ingress is all it takes)
- `domainFilters` — a guardrail: only ever manage records in your zone
- `policy: sync` — create, update **and** delete (remove the Ingress → the record is removed; use `upsert-only` if you never want auto-deletes)
- `registry: txt` + `txtOwnerId` + `txtPrefix` — the **ownership** mechanism: ExternalDNS stamps a TXT record so it only ever touches records it created
- the Cloudflare token, read from the Secret you just made

```bash
helm repo add external-dns-charts https://kubernetes-sigs.github.io/external-dns/
helm repo update
helm install external-dns-controller external-dns-charts/external-dns \
  -n external-dns -f k8s/external-dns-values.yaml

kubectl rollout status -n external-dns deploy/external-dns-controller
kubectl logs -n external-dns deploy/external-dns-controller
```

✅ The logs show it started and connected to Cloudflare with **no errors** (e.g.
"All records are already up to date") — there's no Ingress yet, so there's nothing
to sync. Keep this log open for the next step.

> Source: the official [ExternalDNS + Cloudflare tutorial](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/cloudflare.md).

## 7. Apply the Ingress — and see why the flag matters

The Ingress is the whole point, and it's tiny. See [k8s/ingress.yaml](k8s/ingress.yaml):

- `ingressClassName: traefik-ingress` — **must match the IngressClass Traefik created**, which is named after the Helm release. Get this wrong and Traefik silently ignores the Ingress (see [troubleshooting](#troubleshooting)).
- `rules[].host` — the hostname ExternalDNS turns into DNS
- `backend` — send that host to the `frontend` Service on port 80
- annotation `external-dns.alpha.kubernetes.io/cloudflare-proxied: "false"` — keep the record DNS-only (grey cloud) so `dig`/`curl` reach the load balancer directly

**First, confirm nothing exists yet:**

```bash
dig +short shop.devopsdetours-lab.com        # → (nothing)
```

Also check your Cloudflare dashboard — no `shop` record.

**7a. Apply it — and watch nothing happen (publishedService still off):**

```bash
kubectl apply -f k8s/ingress.yaml
kubectl get ingress -n shop                  # ADDRESS column is EMPTY
kubectl logs -n external-dns deploy/external-dns-controller
# → still "All records are already up to date": the Ingress has no address to read
dig +short shop.devopsdetours-lab.com        # → still (nothing)
```

Notice there's **no error** — nothing crashed. The Ingress simply has no address,
so ExternalDNS has nothing to create. That's the trap: it fails silently.

**7b. Turn the flag on — and watch it come to life:**

`publishedService` tells Traefik to write its load balancer address into every
Ingress's status — exactly what ExternalDNS reads.

```bash
helm upgrade traefik-ingress traefik-charts/traefik \
  -n traefik \
  --set providers.kubernetesIngress.publishedService.enabled=true

kubectl get ingress -n shop -w               # ADDRESS now fills in with the ELB host
```

Within a reconcile interval, ExternalDNS acts:

```bash
kubectl logs -n external-dns deploy/external-dns-controller -f
# ... CREATE: shop.devopsdetours-lab.com          CNAME -> ...elb.amazonaws.com
# ... CREATE: extdns-...shop.devopsdetours-lab.com  TXT   (ownership record)
```

✅ Refresh your Cloudflare dashboard: a **CNAME** and a **TXT** (prefixed `extdns-`)
appear — created for you. One flag was the whole difference.

## 8. Verify it resolves end-to-end

```bash
dig +short shop.devopsdetours-lab.com        # → the ELB hostname
curl -s http://shop.devopsdetours-lab.com | head
```

✅ Open `http://shop.devopsdetours-lab.com` in a browser — the Online Boutique shop
loads, at a real name. The hostname you only declared in YAML now works.

> First resolution can lag a little (ExternalDNS reconcile interval + DNS
> propagation). This repo sets `interval: 10s` in
> [k8s/external-dns-values.yaml](k8s/external-dns-values.yaml) to keep things snappy
> (the chart default is `1m`).

## 9. (Optional) Prove it's a real controller — delete = cleanup

```bash
kubectl delete -f k8s/ingress.yaml
kubectl logs -n external-dns deploy/external-dns-controller -f
# ... DELETE: shop... CNAME ...
# ... DELETE: extdns-...shop... TXT ...
dig +short shop.devopsdetours-lab.com        # → empty again
```

Because `policy: sync`, removing the Ingress removes the records — and only the
records ExternalDNS owned (that's what the TXT record is for). It never touches
anything you created by hand.

## 10. Tear it all down (stop paying for the cluster)

```bash
kubectl delete -f k8s/ingress.yaml --ignore-not-found
kubectl delete -n shop \
  -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/release/kubernetes-manifests.yaml
helm uninstall external-dns-controller -n external-dns
kubectl delete namespace external-dns        # removes the cloudflare-token Secret too
kubectl delete -f k8s/namespace.yaml
helm uninstall traefik-ingress -n traefik

cd terraform && terraform destroy && cd ..   # tear down the cluster
```

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---------|--------------------|
| Ingress `ADDRESS` stays empty | (1) Traefik installed without `publishedService.enabled=true` → add it with `helm upgrade`. (2) `ingressClassName` in `ingress.yaml` doesn't match the IngressClass Traefik created (`kubectl get ingressclass`) — they must match, or Traefik ignores the Ingress entirely. |
| ExternalDNS: `flag 'interval' cannot be repeated` (crash) | Don't set `--interval` under `extraArgs` — the chart already renders it. Use the top-level `interval:` value instead. |
| ExternalDNS logs: auth / 403 errors | Token scopes wrong, or wrong zone → recheck the token (DNS:Edit + Zone:Read on your zone). |
| Record never created | `domainFilters` must match your zone, and the Ingress `host` must be **under** that zone. |
| `dig` empty after a few minutes | Cloudflare cache / propagation. Confirm the record exists in the dashboard; the `cloudflare-proxied: "false"` annotation keeps it DNS-only so it resolves straight to the ELB. |
| App pods stuck `Pending` | Not enough capacity → raise `desired_nodes` / `max_nodes` in [terraform/variables.tf](terraform/variables.tf). |
| `TXT/CNAME conflict` error | Make sure `txtPrefix` is set (it is: `extdns-`) so the TXT name doesn't collide with the CNAME. |
