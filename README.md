# ExternalDNS on EKS with Traefik + Cloudflare

Automatic DNS for Kubernetes, explained step by step. Declare a hostname **once**
in an Ingress, and ExternalDNS keeps a Cloudflare record in sync for you —
completely out of your users' traffic path.

📺 **Watch the video:** <add-video-url>

---

## What you'll build

- An **Amazon EKS** cluster (Terraform)
- **Traefik** as the ingress controller (gets an AWS load balancer)
- **Online Boutique** as the demo app
- **ExternalDNS** watching your Ingress and syncing a Cloudflare `CNAME` (+ a `TXT`
  ownership record)

End result: `shop.<your-domain>` resolves to your app automatically — no DNS
records edited by hand.

## Start here

👉 **[walkthrough.md](walkthrough.md)** — the full step-by-step guide. It lists the
handful of values to change to your own (zone, hostname, cluster) and walks the
whole deploy end to end, including the one Traefik flag everyone gets wrong.

## What's in this repo

| Path | What it is |
|------|------------|
| [walkthrough.md](walkthrough.md) | The hands-on guide — read this first |
| [k8s/](k8s/) | Kubernetes manifests: Ingress, ExternalDNS values, namespace |
| [terraform/](terraform/) | The EKS cluster |
| [slides/](slides/) | The diagrams from the video (PDF) |
| [.env.example](.env.example) | Template for your Cloudflare API token |

## Prerequisites

- An AWS account, plus `terraform`, `kubectl`, `helm`, and the AWS CLI (configured)
- A domain on **Cloudflare** (zone active) and an API token scoped to it
  (**Zone › DNS › Edit** + **Zone › Zone › Read**)

> Your Cloudflare token goes in a local `.env` (gitignored) — never commit it.

## Part of a series

This is **part 1** of a 3-part series:

1. **ExternalDNS** — automatic DNS (this repo)
2. **cert-manager** — automatic HTTPS on the same hostname
3. **The Detour** — hand DNS records from Terraform to ExternalDNS, with zero downtime
