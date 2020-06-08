# Argo

Kustomize templates for Argo into Argo CD managed namespaces.

## Repository layout

### Base

Base folder points to a remote kustomize base for [Argo Events](https://github.com/argoproj/argo-events/tree/master/manifests). It uses a namespaced variant of the manifests.

Additionally all additional generic Roles and RoleBindings are defined here.

### Overlays

Each overlay is tied to a specific namespace on a cluster. Argo CD Application definition then specifies to which cluster the overlay belongs to.

Each overlay defines their user mapping to given `project-argo-events-users` Role by patching the respective resources.

Additionally an overlay can define additional namespace specific resources - usually additional Roles or RoleBindings required in that particular context.

## Usage

If you desire to create a new Argo deployment, simply create a new overlay with a Kustomization file that at minimum speficies the base and the namespace:

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: <NAMESPACE>

resources:
  - ../../base
```

Then create a new Argo CD application with a following definition:

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <NAME>
  namespace: <TARGET_ARGOCD_NAMESPACE>
spec:
  project: <PROJECT>
  source:
    repoURL: https://github.com/AICoE/aicoe-sre
    targetRevision: HEAD
    path: applications/argo-events/overlays/<YOUR_OVERLAY>
  destination:
    server: <CLUSUTER_NAME>
    namespace: <NAMESPACE>
```

## Deployment guide

Run the following command from the root of this repository to deploy
Jupyterhub in a ovelayed environment:

`${target_env}` must much a folder in `overlays`.

```bash
kustomize build overlays/${target_env}/ | oc apply -f -
```

## Build manifests

If you want to build the manifests on your local file system without deploying
them run the following command from the root of this repository:

```bash
mkdir build
kustomize build overlays/${target_env}/ -o build/
```
