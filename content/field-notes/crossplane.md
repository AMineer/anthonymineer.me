---
title: "Crossplane for People Who Already Know Terraform"
date: 2026-08-09
tags: [platform-engineering, crossplane, kubernetes, terraform, iac, homelab]
draft: false
---

## Start here

You already know Terraform. So the fastest way to explain Crossplane is to tell you what it is not: it is not Terraform running inside a cluster. It is a different model, and the difference is the entire reason to care.

Terraform is something you run. You point it at some config, it builds a plan, you apply, it exits. Crossplane is something that runs. You install it into Kubernetes, tell it what infrastructure you want, and a controller keeps reality matching that want forever. Nobody types `apply`. The loop never stops.

That one shift, from "run a tool" to "operate a control plane," is where all the advantages and all the pain come from. Let me walk through what it is, how it stacks up against Terraform when you're honest about it, why it makes self-service almost fall out for free, and how to stand up a minimal version in a homelab this afternoon.

## What it actually is

Crossplane extends the Kubernetes API with custom resources that represent cloud infrastructure. You install a provider (the GCP provider, the AWS provider, and so on), and it brings a pile of new resource types called managed resources: `Bucket`, `Instance`, `Network`, the usual suspects. You write a YAML manifest that says "I want a GCS bucket in US, uniform access on," apply it, and a controller reconciles the real bucket into existence and then watches it.

Watches it is the load-bearing word. Delete that bucket in the cloud console and Crossplane notices the drift and recreates it. Change a setting by hand and it changes it back. Your desired state does not live in a `tfstate` file somewhere in a bucket with a DynamoDB lock. It lives in the cluster's etcd, and a controller enforces it on a loop.

On top of the raw managed resources sits the part that makes Crossplane a platform tool instead of a novelty:

- **XRDs (composite resource definitions)** let you define your own API. Instead of exposing a `Bucket` with its forty knobs, you publish a `PlatformBucket` with the three that your org actually lets people choose.
- **Compositions** say what a `PlatformBucket` is made of: a bucket, plus an IAM policy, plus a label scheme, plus whatever standards you want baked in and non-negotiable.
- **Composition functions** are the logic that assembles all that. This is the real programming surface, and in v2 it is the way you compose. The old native patch-and-transform style was deprecated back in v1.17.

A couple of v2 specifics worth knowing if you're coming from old tutorials, because v2 changed the shape of things. Composite resources and managed resources are namespaced by default now. Claims, the old cluster-scoped `XRC` layer, are gone, which removes a whole confusing tier. An XR can compose any Kubernetes resource, not just cloud managed resources, so a single abstraction can stitch together a bucket, a deployment, and a third-party CRD. And managed resource definitions let you activate only the resource types you need instead of letting a provider carpet-bomb your API server with a few thousand CRDs.

## Crossplane vs Terraform, without the marketing

This is the part people get wrong in both directions, so here is the honest version.

Where Crossplane wins:

Continuous reconciliation is real and it is the headline feature. Terraform only knows about drift when someone remembers to run `plan`. Crossplane knows within its reconcile interval and fixes it without anyone asking. For infrastructure that must stay a certain way, that is a genuinely different guarantee.

It is a real API, which is the self-service superpower I'll come back to. Anything that can talk to the Kubernetes API can request infrastructure, with RBAC and namespaces giving you actual multi-tenancy instead of "we all share one state file and pray."

Composition lets you publish a narrow, opinionated interface and hide the sharp edges. That is the difference between self-service that stays clean and self-service that becomes a landfill.

It is GitOps-native by default. Your claims are just manifests, so Argo or Flux sync them like anything else, and the reconciliation model and the GitOps model actually agree with each other instead of fighting.

Where Crossplane hurts, and you need to hear this before you commit:

You are now running a control plane. That is an always-on system to patch, monitor, upgrade, and debug at 2am. Terraform is a binary that runs in CI and then stops existing. Crossplane never stops existing. That is not free.

Drift correction is a double-edged sword. Crossplane will fight anything that touches its resources, including you, including another tool, including that one emergency console change someone made during an incident. If you are not fully committed to Crossplane owning a resource, it will cause conflicts, not prevent them.

Debugging compositions is its own skill you have to build. When a composition function misbehaves, you are reading Kubernetes events, controller logs, and traces. You do not get the clean, readable `terraform plan` diff that tells you exactly what is about to change. The feedback loop is worse, and pretending otherwise sets your team up to hate the tool.

Provider coverage and maturity vary. For any given resource, the Terraform provider is often more complete and better documented than the Crossplane equivalent. Check before you assume parity.

And your state lives in etcd now. Back it up and have a recovery story, because "restore the control plane" is a sentence you want to have rehearsed before you need it.

The honest conclusion is that this is not Terraform-but-better. It is a different tool for a different job. Terraform is excellent at provisioning a substrate on demand behind a reviewable plan. Crossplane is excellent at being a continuously-enforced, self-service API for infrastructure. Most shops that run Crossplane well also still run Terraform: Terraform lays down the landing zone, the network, the cluster itself, and Crossplane runs on top as the layer developers actually touch. Reach for that split before you reach for a rip-and-replace. Anyone selling you Crossplane as a Terraform killer is selling you a second on-call rotation.

## Why this makes self-service almost free

Here is the thing that clicks once you sit with it. Because Crossplane turns infrastructure into a Kubernetes API, putting a portal in front of it is trivial, since a Kubernetes API is the easiest thing in the world to automate against.

A developer portal like Port can expose a "request a bucket" form whose only real job is to create one Crossplane resource. The developer sees three fields. The platform sees a namespaced object with RBAC, labels, and an audit trail. The controller does the provisioning and then keeps the result correct forever, with zero portal involvement after that first `create`.

Compare that to wiring the same self-service flow on raw Terraform. Now the portal has to trigger a pipeline, manage state locking, babysit the run's lifecycle, capture outputs, and handle failure and retry. With Crossplane the "run" is just the controller doing what controllers do, and the portal's entire responsibility shrinks to writing a single object into the cluster.

The deeper integration, wiring Port actions to a GitOps commit path and surfacing the resulting claims back into the catalog, is its own post and I'll write it. The point for today is narrower: the control-plane model plus the Kubernetes API means self-service is not a feature you build, it is a property you get.

## A minimal homelab you can build this afternoon

You do not need a cloud platform team to try this. You need a throwaway cluster and about twenty minutes.

I use k3d because it spins up in seconds and I do not feel anything when I delete it. kind works fine too. The point is that this cluster is disposable, so you experiment without fear.

Install the Crossplane core with Helm:

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system --create-namespace
```

Install a provider. The GCP provider is a family now, so for buckets you want the storage member rather than one giant monolith. Pin to whatever the current tag is on the Upbound Marketplace:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp-storage
spec:
  # check the marketplace for the current version tag
  package: xpkg.upbound.io/upbound/provider-gcp-storage:v2.5.0
```

Give it credentials. For a homelab, the pragmatic path is a GCP service account key in a Secret. In production you would use Workload Identity Federation instead of a static key, which is exactly the OIDC path I used in the Port and GitLab build, but for a lab a scoped SA key is fine. Just do not commit it to a repo:

```bash
kubectl create secret generic gcp-creds \
  -n crossplane-system \
  --from-file=creds=./gcp-sa-key.json
```

Point a ProviderConfig at that secret:

```yaml
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: your-project-id
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: gcp-creds
      key: creds
```

Now create your first managed resource. This is the classic cluster-scoped form, which works everywhere. In v2 the namespaced equivalent lives under the `.m` API group suffix as providers add support, so once your GCP provider version supports it you would move to `storage.gcp.m.upbound.io` and give the resource a namespace:

```yaml
apiVersion: storage.gcp.upbound.io/v1beta1
kind: Bucket
metadata:
  generateName: fieldnotes-lab-
  labels:
    managed-by: crossplane
spec:
  forProvider:
    location: US
    storageClass: STANDARD
    uniformBucketLevelAccess: true
  providerConfigRef:
    name: default
```

Apply it, then watch it reconcile:

```bash
kubectl get buckets -w
```

When `SYNCED` and `READY` both flip to `True`, the bucket exists in GCP. Now do the one thing that makes the whole model click: go into the cloud console and delete the bucket by hand. Then watch your terminal. Thirty seconds later, Crossplane has noticed it is gone and built it again.

That single gesture is the entire pitch. Terraform would sit there none the wiser until the next `plan`. Crossplane treated your manual deletion as drift and corrected it, because as far as it is concerned, the manifest is the truth and the cloud is just a cache of that truth.

## Should you use it

If you already live in Kubernetes and you want continuously-enforced, self-service infrastructure with a real API and real RBAC, Crossplane earns its keep and the operational cost is worth paying. If you stand up a landing zone once a quarter and move on, keep using Terraform and do not let anyone make you feel behind for it. The right answer for most teams is both, with each doing the job it is actually good at.

Next in this thread: wiring this exact Crossplane setup into Port so a developer gets a form and the platform gets a namespaced, audited, self-healing resource, with a GitOps commit path in between so nothing lands without a trail.