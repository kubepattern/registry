# KubePattern Project: Pattern as Code Registry

Welcome to the official public registry for the KubePattern project. This repository contains a collection of "Pattern as Code" definitions.

## What is "Pattern as Code"?

**Pattern as Code** is a declarative approach to defining Kubernetes smells.

These patterns are Kubernetes Custom Resources written in YAML files, each describing the unique characteristics, resources, and relationships of a specific configuration bad practice.

The [KubePattern](https://kubepattern.dev) analysis tool uses these definitions to analyze your Kubernetes cluster. By comparing the cluster's state against the patterns defined in this registry, KubePattern can identify existing bad configurations and link to remediation documentation comprehensive of practical suggestions for enforcing best practices.

## Registry layout

Patterns are grouped by the application they analyze. Each folder holds the Patterns in `patterns/`, a `platform.yaml` that describes the application and, where available, documentation in `doc/` and test manifests in `tests/`.

| Application | Patterns |
|---|---|
| [Kubernetes](kubernetes/patterns) | `missing-liveness-probe`, `missing-resource-limits`, `naked-pod`, `service-selects-no-pod` |
| [Argo CD](argocd/patterns) | `argocd-applicationset-generates-nothing`, `argocd-appproject-empty` |
| [Cluster API](capi/patterns) | `capi-machinetemplate-orphan` |
| [cert-manager](cert-manager/patterns) | `certmanager-clusterissuer-not-used` |
| [CloudNativePG](cnpg/patterns) | `cnpg-cluster-without-scheduledbackup`, `cnpg-pooler-name-clashes-with-cluster` |
| [Crossplane](crossplane/patterns) | `crossplane-composition-without-xrd`, `crossplane-function-not-used` |
| [External Secrets](external-secrets/patterns) | `eso-clustersecretstore-not-used`, `eso-secretstore-not-used` |
| [Flux](flux/patterns) | `flux-ocirepository-not-used` |
| [Kargo](kargo/patterns) | `kargo-warehouse-not-requested` |
| [KubeVela](kubevela/patterns) | `kubevela-componentdefinition-not-used`, `kubevela-traitdefinition-not-used`, `kubevela-builtin-componentdefinition-not-used`, `kubevela-builtin-traitdefinition-not-used` |
| [Krateo](krateo/patterns) | `krateo-compositiondefinition-unused`, `krateo-navmenuitem-page-missing`, `krateo-restaction-endpoint-missing`, `krateo-restaction-manager-missing`, `krateo-restaction-not-used`, `krateo-rolebinding-clusterrole-missing`, `krateo-rolebinding-subject-sa-missing`, `krateo-widget-manager-missing`, `krateo-yamlviewer-apiref-dangling`, `page-not-referenced`, `panel-not-referenced`, `paragraph-not-referenced` |

Some Patterns use the DSL constructs of the limitations update (element anchors `[@]`, defaults, value transforms, quoted keys, `kinds`, kind wildcards, `fromTarget`, criteria on `selects`/`selectedBy`) or `spec.for`: they need a KubePattern engine built from the `dev` branch.

## Metadata

Every Pattern in `patterns/` carries two labels and one annotation. They play no part in detection: the [Smell Map](https://kubepattern.dev/smell-map) of the KubePattern site is built from them, so a Pattern added here appears on the map at the next deploy.

```yaml
metadata:
  name: eso-secretstore-not-used
  labels:
    kubepattern.dev/platform: external-secrets   # the name of the folder
    kubepattern.dev/family: unused                # unused | dangling | companion | misconfig | single
  annotations:
    kubepattern.dev/summary: "A SecretStore that no ExternalSecret or PushSecret in its namespace uses, still holding provider credentials."
```

| Family | What the Pattern reports |
|---|---|
| `unused` | A definition exists, but nothing refers to it. |
| `dangling` | A resource points at something that does not exist. |
| `companion` | A resource lacks the sibling that should accompany it. |
| `misconfig` | A relation exists that should not. |
| `single` | A single resource lacks a field it should have, with no relationship involved. |

The summary is one sentence, in English, that says what the Pattern flags and why it matters.

Each application folder has a `platform.yaml`:

```yaml
name: "External Secrets"   # display name
role: "Secrets sync"       # what the application is, in a few words
version: "2.11.0"          # optional: version the Patterns were written and tested against
note: "..."                # optional: extra context shown on the map
```

The site build fails, and the site keeps its previous version, if a Pattern lacks one of these fields or its folder lacks a `platform.yaml`.

## Learn to write Pattern as Code
Check out the official documentation: [KubePattern Documentation](https://kubepattern.dev/docs)
