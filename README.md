# httpbin
Project with a Helm Chart deploying httpbin service, meant to be used with Flux2.

### Prerequisites

Kubernetes cluster with [Flux](https://github.com/fluxcd/flux) installed and `fluxctl`.

### Git repository
`GitRepository` defines the source that contains the chart.
Trusted sources will be registered by a cluster administrator by creating the resources in the flux-system namespace.

```
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: httpbin
spec:
  interval: 30s # Defines at which interval the Git repository contents are fetched
  url: https://github.com/gonzalobarbitta/httpbin
  ref:
    branch: main
```

Or, using `fluxctl`

```
flux create source git httpbin \
  --url=https://github.com/gonzalobarbitta/httpbin \
  --branch=main \
  --interval=30s \
  --export > ./source.yaml
```

### Kustomization
Then, we create a `kustomization` for synchronizing the manifests on the cluster.

```
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: httpbin
spec:
  interval: 1m
  path: ./namespaces
  prune: true
  sourceRef:
    kind: GitRepository
    name: httpbin
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: httpbin-releases
spec:
  interval: 1m
  dependsOn:
    - name: httpbin-namespaces
  path: ./releases
  prune: true
  sourceRef:
    kind: GitRepository
    name: httpbin
```

Or, using `fluxctl`

```
flux create kustomization httpbin-namespaces \
  --source=httpbin \
  --path="./namespaces" \
  --prune=true \
  --validation=client \
  --interval=1m \
  --export > ./kustomization-namespaces.yaml
```

```
flux create kustomization httpbin-releases \
  --source=httpbin \
  --path="./releases" \
  --prune=true \
  --validation=client \
  --interval=1m \
  --depends-on=httpbin-namespaces \
  --export > ./kustomization-releases.yaml
```

### Helm Release

One of these manifests will be the Helm Release, which references the chart source (Git repository).

 ```
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: httpbin
  namespace: httpbin # Namespace where the chart will be installed
spec:
  interval: 1m
  targetNamespace: httpbin # Namespace in which the chart components will be installed
  chart:
    spec:
      chart: ./charts/httpbin # Path inside the git source where chart is located
      sourceRef:
        kind: GitRepository
        name: httpbin
      interval: 1m
 ```