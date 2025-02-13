# Flux

Coursework for [Flux CD Mastery: Automating Kubernetes with GitOps](https://www.udemy.com/course/flux-cd-mastery-automating-kubernetes-with-gitops/).

## Pre-reqs and Setup

Ended up using [Rancher Desktop](https://docs.rancherdesktop.io/references/rdctl-command-reference/#rdctl-start)

```sh
# boot rancher desktop
rdctl start

# graceful shutdown
rdctl shutdown
```

## Bootstrapping Flux

[Bootstrapping](https://fluxcd.io/flux/installation/bootstrap/github/#github-pat) initializes the flux components in the k8s cluster's `flux-system` ns.

```sh
export GITHUB_TOKEN=<your-gh-token>

flux bootstrap github \
  --owner=cbrown-ltk \
  --private=false --personal=true \
  --repository=udemy-flux \
  --branch=main \
  --path=clusters/local \
  --token-auth
```

Exploring the components deployed:

```sh
kubectl config set-context --current --namespace flux-system

# misc components (see clusters/local/flux-system/gotk-components.yaml)
kubectl get secret flux-system -o=yaml | yq .data
kubectl get clusterrolebindings | grep flux

# State is retrieves thru sources
flux get sources git

# NAME                     	REVISION          	SUSPENDED	READY	MESSAGE
# gitrepository/flux-system	main@sha1:55454cbb	False    	True 	stored artifact for revision 'main@sha1:55454cbb'

# these can also be retrieved via kubectl
kubectl get gitrepo

# NAME          URL                                          AGE     READY   STATUS
# flux-system   ssh://git@github.com/cbrown-ltk/udemy-flux   5m22s   True    stored artifact for revision 'main@sha1:2e1b73b26a6c2f0fd39de6542df9172ccb65d7c4'
```

## Deploying a FluxCD kustomize app

Deploying a kustomize-based [podinfo](https://github.com/stefanprodan/podinfo), following a [monorepo structure](https://fluxcd.io/flux/guides/repository-structure/):

```sh
# helper:
# alias kk_init='echo -e "apiVersion: kustomize.config.k8s.io/v1beta1\nkind: Kustomization\nresources:\n  - ./deployment.yaml\n" > kustomization.yaml'
# ref: https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/

# lets deploy an app "Podinfo" (TODO -- could also try apps/podinfo/{base,local,env1})
mkdir -p apps/{base,local}/podinfo

# .
# └── apps
#     ├── base           # the set of "Base" apps (available off-the-shelf)
#     │   └── podinfo
#     └── local          # deployed instances
#         └── podinfo
```

First, create the "base" app-template in `apps/base/{appName}` . This will be leveraged (and values possibly customized) by app-instances in `apps/{cluster}/{appName}`:

```sh
cd apps/base/podinfo

# Scaffold a source from a public Git repository master branch
flux create source git podinfo --export \
  --url=https://github.com/stefanprodan/podinfo \
  --branch=master \
  --interval 5m0s > repo.yaml

# Scaffold a Flux Kustomization resource from a source at a given path
flux create kustomization podinfo --export \
    --source=GitRepository/podinfo \
    --path="./kustomization" \
    --prune=true \
    --interval=5m0s \
    --wait=true > release.yaml

# a vanilla k8s namespace
echo -e "apiVersion: v1\nkind: Namespace\nmetadata:\n  name: default" > namespace.yaml

# generate a kustomize.k8s.io with the resources above
kk_init
yq eval '.resources = ["./namespace.yaml","./release.yaml","./repo.yaml"]' -i kustomization.yaml
```

Next, create the app-instance for `clusters/local`:

```sh
# navigating back to the app-instances, pointing back to the base instance
cd ../../local/podinfo
kk_init
yq eval '.resources = ["../../base/podinfo"]' -i kustomization.yaml
```

Finally, tell Flux to deploy collection of app-instances in `apps/local`:

```sh
cd ../../../clusters/local

# Create a FluxCD Kustomization resource from a source at a given path
flux create kustomization apps \
  --export \
  --source=GitRepository/flux-system \
  --path="./apps/local" \
  --prune=true \
  --interval=10m0s \
  --wait=true \
  --health-check-timeout=5m0s > apps.yaml
```

Finally, `git commit` and push up to begin the reconciliation process. To watch progress:

```sh
# a FluxCD Kustomization is similar to k8s kustomization, but uses FluxCD concepts (e.g, gitrepository, oci, helmRelease, etc)
flux get kustomizations

# List GitRepository sources and their status
flux get sources git

# watch the Kustomization
flux get ks # --watch

# examine the deployed app
kn default
kubectl port-forward deploy/podinfo -- 9898:9898

# in a separate tab
curl http://localhost:9898

# {
#   "hostname": "podinfo-9d8789c44-twkzn",
#   "version": "6.7.1",
#   "revision": "6b7aab8a10d6ee8b895b0a5048f4ab0966ed29ff",
#   "color": "#34577c",
#   "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
#   "message": "greetings from podinfo v6.7.1",
#   "goos": "linux",
#   "goarch": "arm64",
#   "runtime": "go1.23.2",
#   "num_goroutine": "8",
#   "num_cpu": "2"
# }
```

## Deploying a Flux HelmRelease app

Example:
- [HelmRelease.fluxcd.io](https://github.com/fluxcd/flux2-kustomize-helm-example/blob/main/apps/base/podinfo/release.yaml)
- Which uses the Helm repo [stefanprodan.github.io/podinfo](https://github.com/stefanprodan/podinfo/tree/master?tab=readme-ov-file#helm)

```sh
# Create a source for an HTTPS public Helm repository (a Flux "HelmRepository")
# flux create source helm --help | grep Examples -A29
flux create source helm stefanprodan \
    --namespace=flux-system \
    --url=https://stefanprodan.github.io/podinfo \
    --interval=10m \
    --export

# be mindful here of "--namespace", as in "Storage Namespace" versus "Target Namespace"
# https://fluxcd.io/flux/components/helm/helmreleases/#target-namespace
HELM_REPO="HelmRepository/stefanprodan.podinfo" # Different namespace
HELM_REPO="HelmRepository/stefanprodan"
HELM_REPO="HelmRepository/stefanprodan.flux-system" # same as above

# Create the HelmRelease
flux create helmrelease podinfo \
  --namespace=flux-system \
  --target-namespace=pi \
  --create-target-namespace=true \
  --source=$HELM_REPO \
  --release-name=podinfo \
  --chart=podinfo \
  --chart-version=">6.7.0" \
  --values=apps/local/podinfo-chart/values.yaml \
  --export > apps/local/podinfo-chart/release.yaml
```
