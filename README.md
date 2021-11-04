# EKS GitOps Workshop

For this example we assume a scenario with two clusters: dev and production.
The end goal is to leverage Flux and Kustomize to manage both clusters while minimizing duplicated declarations.

We will configure Flux to install, test and upgrade a demo app using
`HelmRepository` and `HelmRelease` custom resources.
Flux will monitor the Helm repository, and it will automatically
upgrade the Helm releases to their latest chart version based on semver ranges.

## Prerequisites

You will need a Kubernetes cluster version 1.16 or newer and kubectl version 1.18.
For a quick local test, you can use [Kubernetes kind](https://kind.sigs.k8s.io/docs/user/quick-start/).
Any other Kubernetes setup will work as well though.

In order to follow the guide you'll need a GitHub account and a
[personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)
that can create repositories (check all permissions under `repo`).

Install the Flux CLI on MacOS and Linux using Homebrew:

```sh
brew install fluxcd/tap/flux
```

Or install the CLI by downloading precompiled binaries using a Bash script:

```sh
curl -s https://fluxcd.io/install.sh | sudo bash
```

## Repository structure

The Git repository contains the following top directories:

- **apps** dir contains Helm releases with a custom configuration per cluster
- **infrastructure** dir contains infra tools such as NGINX ingress controller and Helm repository definitions
- **clusters** dir contains the Flux configuration per cluster

```
├── apps
│   ├── base
│   ├── production 
│   └── dev
├── infrastructure
│   ├── nginx
│   ├── redis
│   └── sources
└── clusters
    ├── production
    └── dev
```

The apps configuration is structured into:

- **apps/base/** dir contains namespaces and Helm release definitions
- **apps/production/** dir contains the production Helm release values
- **apps/dev/** dir contains the dev values

```
./apps/
├── base
│   └── podinfo
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       └── release.yaml
├── production
│   ├── kustomization.yaml
│   └── podinfo-patch.yaml
└── dev
    ├── kustomization.yaml
    └── podinfo-patch.yaml
```

In **apps/base/podinfo/** dir we have a HelmRelease with common values for both clusters:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo
spec:
  releaseName: podinfo
  chart:
    spec:
      chart: podinfo
      sourceRef:
        kind: HelmRepository
        name: podinfo
        namespace: flux-system
  interval: 5m
  values:
    cache: redis-master.redis:6379
    ingress:
      enabled: true
      annotations:
        kubernetes.io/ingress.class: nginx
```

In **apps/dev/** dir we have a Kustomize patch with the dev specific values:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo
spec:
  chart:
    spec:
      version: ">=1.0.0-alpha"
  test:
    enable: true
  values:
    ingress:
      hosts:
        - host: podinfo.dev
```

Note that with ` version: ">=1.0.0-alpha"` we configure Flux to automatically upgrade
the `HelmRelease` to the latest chart version including alpha, beta and pre-releases.

In **apps/production/** dir we have a Kustomize patch with the production specific values:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo
spec:
  chart:
    spec:
      version: ">=1.0.0"
  values:
    ingress:
      hosts:
        - host: podinfo.production
```

Note that with ` version: ">=1.0.0"` we configure Flux to automatically upgrade
the `HelmRelease` to the latest stable chart version (alpha, beta and pre-releases will be ignored).

In **infrastructure/dev/** dir we have infrastructure toold with dev specific values:

```
./infrastructure/dev
├── nginx
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── release.yaml
├── redis
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── release.yaml
└── sources
    ├── bitnami.yaml
    ├── kustomization.yaml
    └── podinfo.yaml
```

In **infrastructure/production/** dir we have infrastructure toold with dev specific values:
```
./infrastructure/production
├── nginx
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── release.yaml
├── redis
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── release.yaml
└── sources
    ├── bitnami.yaml
    ├── kustomization.yaml
    └── podinfo.yaml
```

In **infrastructure/*/sources/** dir we have the Helm repositories definitions:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: podinfo
spec:
  interval: 5m
  url: https://stefanprodan.github.io/podinfo
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: bitnami
spec:
  interval: 30m
  url: https://charts.bitnami.com/bitnami
```

Note that with ` interval: 5m` we configure Flux to pull the Helm repository index every five minutes.
If the index contains a new chart version that matches a `HelmRelease` semver range, Flux will upgrade the release.

## Bootstrap dev and production

The clusters dir contains the Flux configuration:

```
./clusters/
├── production
│   ├── apps.yaml
│   └── infrastructure.yaml
└── dev
    ├── apps.yaml
    └── infrastructure.yaml
```

In **clusters/dev/** dir we have the Kustomization definitions:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m0s
  dependsOn:
    - name: infrastructure
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/dev
  prune: true
  validation: client
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/dev
```

Note that with `path: ./apps/dev` we configure Flux to sync the dev Kustomize overlay and 
with `dependsOn` we tell Flux to create the infrastructure items before deploying the apps.

Fork this repository on your personal GitHub account and export your GitHub access token, username and repo name:

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=<repository-name>
```

Verify that your dev cluster satisfies the prerequisites with:

```sh
flux check --pre
```

Set the kubectl context to your dev cluster and bootstrap Flux:

```sh
flux bootstrap github \
    --context=dev \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --private=false \
    --personal \
    --path=clusters/dev
```

The bootstrap command commits the manifests for the Flux components in `clusters/dev/flux-system` dir
and creates a deploy key with read-only access on GitHub, so it can pull changes inside the cluster.

Watch for the Helm releases being install on dev:

```console
$ watch flux get helmreleases --all-namespaces 
NAMESPACE	NAME   	REVISION	SUSPENDED	READY	MESSAGE                          
nginx    	nginx  	5.6.14  	False    	True 	release reconciliation succeeded	
podinfo  	podinfo	5.0.3   	False    	True 	release reconciliation succeeded	
redis    	redis  	11.3.4  	False    	True 	release reconciliation succeeded
```

Verify that the demo app can be accessed via ingress:

```console
$ kubectl -n nginx port-forward svc/nginx-ingress-controller 8080:80 &

$ curl -H "Host: podinfo.dev" http://localhost:8080
{
  "hostname": "podinfo-59489db7b5-lmwpn",
  "version": "5.0.3"
}
```

Bootstrap Flux on production by setting the context and path to your production cluster:

```sh
flux bootstrap github \
    --context=production \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --private=false \
    --branch=main \
    --personal \
    --path=clusters/production
```

Watch the production reconciliation:

```console
$ watch flux get kustomizations
NAME          	REVISION                                        READY
apps          	main/797cd90cc8e81feb30cfe471a5186b86daf2758d	True
flux-system   	main/797cd90cc8e81feb30cfe471a5186b86daf2758d	True
infrastructure	main/797cd90cc8e81feb30cfe471a5186b86daf2758d	True
```
