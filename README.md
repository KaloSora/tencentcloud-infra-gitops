# Tencent Cloud Infrastructure GitOps

GitOps-managed Tencent Cloud infrastructure powered by Crossplane.

This repository is intended to hold declarative configuration for **new Tencent Cloud resources** managed by Crossplane. It complements the existing Terraform-managed Kubernetes foundation and application delivery repositories.

## Current Status

The repository contains the `infrastructure` AppProject and three manual-sync Applications for a pinned Tencent Cloud Provider (`v0.8.6`), ProviderConfig, and a private development COS bucket. Runtime compatibility and cloud operations still require validation on the target cluster.

See [the COS pilot runbook](docs/COS_PILOT.md) for deployment order, readiness checks and cleanup. Replace the bucket's `REPLACE_APPID` placeholder before its first sync. The Terraform foundation supplies the credential Secret and uses `ap-guangzhou` for this pilot; no one-time Secret script is required.

## Architecture

```mermaid
flowchart TD
    Operator[Operator] -->|Reviewed Terraform plan and apply| Foundation[Cloud and Kubernetes foundation]
    Foundation --> Argo[Argo CD]
    Foundation --> Crossplane[Crossplane]
    InfraRepo[Infrastructure GitOps repository] -->|Pull and sync| Argo
    AppRepo[Application GitOps repository] -->|Pull and sync| Argo
    Argo --> InfraObjects[Crossplane resource declarations]
    Argo --> Workloads[Application Kubernetes manifests]
    InfraObjects --> Crossplane
    Crossplane --> Provider[Tencent Cloud Provider]
    Provider <-->|Observe and reconcile through cloud APIs| Cloud[New Tencent Cloud resources]
```

Terraform creates the initial foundation through an operator-triggered workflow. Argo CD synchronizes Git declarations into Kubernetes. Crossplane and its Provider reconcile the corresponding external cloud resources.

Crossplane does not pull this Git repository itself. Argo CD performs that step. Resource reconciliation depends on Provider capabilities, cloud permissions, and the management policies configured for each resource.

### Related Repositories

| Repository | Responsibility |
| --- | --- |
| [tencentcloud-k8s](https://github.com/KaloSora/tencentcloud-k8s) | Terraform-managed cloud resources, self-hosted Kubernetes foundation, and platform bootstrap |
| [tencentcloud-k8s-app](https://github.com/KaloSora/tencentcloud-k8s-app) | Application source, build pipeline, and promotion logic |
| [tencentcloud-k8s-gitops](https://github.com/KaloSora/tencentcloud-k8s-gitops) | Application deployment declarations and application Argo CD configuration |
| [tencentcloud-infra-gitops](https://github.com/KaloSora/tencentcloud-infra-gitops) | Crossplane configuration and declarations for additional cloud infrastructure |

## Ownership Boundaries

Each cloud resource must have one active write manager.

- Existing CVMs, networking, storage, and other Terraform-owned resources remain under Terraform unless an explicit migration is planned.
- New resources declared here become Crossplane-owned after reconciliation is enabled.
- Crossplane installation and initial Argo CD bootstrap are intended to remain part of the Terraform foundation. This repository will own Provider configuration and subsequent infrastructure declarations.
- An existing Terraform-owned network may be referenced by a Crossplane-owned resource without transferring ownership of that network.
- Creating a resource object with an existing cloud ID can adopt a live resource. Imports require a separate review of ownership, desired configuration, and deletion behavior.

These boundaries apply to Kubernetes objects and Helm releases too: Terraform and Argo CD must not manage the same object or release concurrently.

## Repository Structure

`argocd`, `platform/providers`, `platform/provider-configs`, `environments/dev/cos` and `docs` exist. Compositions and the higher environment directories remain planned.

```text
.
├── README.md
├── argocd/
│   ├── projects/                  # Dedicated infrastructure AppProject
│   └── applications/              # Applications for platform and environment configuration
├── platform/
│   ├── providers/                 # Pinned Provider package declarations
│   ├── provider-configs/          # Authentication references; no credential values
│   └── compositions/              # Optional platform APIs, definitions, and compositions
├── environments/
│   ├── dev/                      # Initial resource pilot
│   ├── uat/
│   ├── preprod/
│   └── prod/
└── docs/                         # Resource ownership, onboarding, and recovery runbooks
```

Argo CD Applications should target specific configuration directories. Do not recursively synchronize the whole repository: examples and environment-specific resources must not be deployed unintentionally.

Environment directories organize declarations; they do not by themselves isolate Tencent Cloud accounts, permissions, or cloud resources. Kubernetes scope also depends on the selected Provider's CRDs.

## Bootstrap and First Resource

1. **Prepare the foundation.** Use the infrastructure repository to provision Kubernetes and install Argo CD and Crossplane. Keep an execution path outside this cluster for rebuilding the foundation.
2. **Select and pin versions.** Validate the Tencent Cloud Provider's compatibility with Crossplane and Kubernetes. Confirm that the required resource types, fields, and lifecycle operations are supported. Avoid floating `latest` versions.
3. **Provide cloud credentials outside Git.** The Terraform foundation now creates `crossplane-system/tencentcloud-credentials` when Crossplane is enabled, using its existing `secret_id` and `secret_key` inputs. The `credentials` key contains JSON with those fields and `region`; only reference this Secret from ProviderConfig in Git. A separate one-time script is no longer required. Credentials are stored in Terraform state, so protect backend and plan access. The foundation's `crossplane_region` input defaults to inheriting its region, with the development configuration explicitly selecting `ap-guangzhou`. Terraform owns this Secret: do not also declare or update it through Argo CD or a script. Existing script-created Secrets require a reviewed one-time Terraform import before apply; see the foundation's `modules/k8s_cicd/CROSSPLANE.md`. Review the supplied cloud identity's permissions and limit them to the intended operations.
4. **Configure infrastructure GitOps access.** Create a dedicated Argo CD AppProject with explicit repository, destination, and resource permissions. The existing application `business-apps` project only permits Deployments and Services and is not suitable for Crossplane resources.
5. **Bootstrap platform configuration.** Synchronize Provider declarations first. Wait for Provider installation, controller health, and CRD availability before applying ProviderConfig or dependent resource objects. Establish explicit readiness gates between these stages.
6. **Start with an isolated development resource.** Use a new test COS bucket, outside the Terraform state backend and management-cluster backup storage. Add its validated declaration in a PR and review its permissions, lifecycle, and expected cost.
7. **Verify the complete lifecycle.** Check creation, a supported update, drift correction on an enforced mutable field, and controller restart recovery. Validate deletion or retention behavior using disposable resources before broader adoption.

The foundation pins Crossplane core to `1.20.3` as an initial candidate for the currently configured Kubernetes `1.28.8`; Helm rendering does not establish runtime compatibility or current upstream support. Validate the version combination on the target cluster before adding Providers.

Merge the two `argocd/` entry points into `main` before deploying the foundation bootstrap. The bootstrap reads `argocd/projects` and `argocd/applications` separately, with automatic self-healing enabled and pruning disabled. Here, only the bootstrap Applications are automatic; future Applications that reconcile cloud resources require separate review before enabling automatic sync. Private repositories need a dedicated Argo CD read credential outside Git.

The `infrastructure` AppProject allows `Provider`, `DeploymentRuntimeConfig`, Tencent Cloud `ProviderConfig`, and COS `Bucket` as cluster resources, and no namespaced resources. The credential Secret remains owned by Terraform. Extend the explicit allowlist only when adding further validated APIs. Terraform owns the Crossplane Helm release and namespace; do not declare them here.

Render the three manual-sync Applications locally with:

```sh
kubectl kustomize argocd/applications
```

Three Applications are expected. Sync them in the order described in the COS pilot runbook. Do not add Applications targeting nonexistent directories or rely solely on child-Application sync waves to wait for Provider CRDs.

## Change Workflow

```text
Create branch
    → Add or update resource declarations
    → Validate against the selected Provider schemas
    → Review ownership, cloud impact, and lifecycle policies
    → Merge approved PR
    → Argo CD synchronizes Kubernetes objects
    → Crossplane Provider reconciles cloud resources
    → Verify actual readiness and application connectivity
```

Before enabling automatic sync, configure branch protection, required reviews, and appropriate validation checks. A merge may authorize billable cloud changes once automatic reconciliation is active.

Git merge and Argo CD synchronization are not sufficient evidence that a cloud resource is usable. Verify the relevant resource conditions, external resource ID, and actual cloud configuration. Dependent applications should wait for infrastructure readiness and required connection Secrets; commit order alone does not enforce readiness.

Console changes to fields enforced by Crossplane may be reverted during reconciliation. Make durable changes through Git. Infrastructure rollback requires reviewing the resulting cloud operation; reverting YAML is not a database restore and may not reverse an immutable-field change.

## State and Recovery

Crossplane does **not** require a separate COS Terraform state backend. Its durable management information is represented by Kubernetes objects, including resource specifications, status, identity annotations, and associated Secrets, persisted by the Kubernetes control plane.

The Tencent Cloud Provider uses Upjet, which handles Terraform-related runtime state internally. This does not require configuring another Terraform CLI backend for this repository.

The existing Terraform foundation continues using its own state backend. Crossplane recovery instead requires a tested backup and recovery process for the management cluster and relevant Kubernetes objects and Secrets.

Git stores desired declarations but does not contain all runtime identity information or credentials. Reapplying Git alone must not be assumed to safely reconnect every existing cloud resource. Document restore or re-adoption procedures using external resource IDs before relying on them.

If COS is used for management-cluster backups, provision that backup storage independently of the control plane being recovered. A dedicated backup bucket is backup storage, not a Crossplane Terraform state backend.

## Security and Deletion Controls

- Never commit cloud keys, kubeconfigs, plaintext or base64-encoded credential Secrets, private keys, or connection passwords.
- Separate application and infrastructure Argo CD permissions. Restrict access to cloud credentials and connection Secrets.
- Review Argo CD pruning, Application deletion propagation, Crossplane management policies, and deletion policies together. Removing a Kubernetes resource object can result in deletion of its external cloud resource.
- Select and test retention behavior for persistent data. Do not treat disabled Argo CD pruning as complete deletion protection.
- Confirm Provider support before relying on `managementPolicies`, including observe-only operation. Observation also needs a separate alerting design if drift notifications are required.
- Test existing-resource adoption separately; do not remove Terraform ownership until a reviewed handover and recovery procedure are ready.

## Implementation Roadmap

- [ ] Validate a pinned Crossplane and Tencent Cloud Provider combination.
- [ ] Implement dedicated credentials and infrastructure Argo CD access.
- [ ] Add staged bootstrap with Provider and CRD readiness checks.
- [ ] Manage a disposable development COS bucket through GitOps.
- [ ] Validate supported updates, drift correction, and deletion or retention behavior.
- [ ] Add manifest validation and required PR checks.
- [ ] Add reconciliation failure alerts and resource ownership records.
- [ ] Document and exercise management-cluster recovery.
- [ ] Introduce Compositions and custom platform APIs when reusable requirements emerge.

## References

- [Crossplane overview](https://docs.crossplane.io/latest/whats-crossplane/)
- [Crossplane managed resources and lifecycle policies](https://docs.crossplane.io/latest/managed-resources/managed-resources/)
- [Importing existing resources](https://docs.crossplane.io/latest/guides/import-existing-resources/)
- [Tencent Cloud Crossplane product overview](https://cloud.tencent.com/document/product/1763/105824)
- [Tencent Cloud Provider source](https://github.com/crossplane-contrib/provider-tencentcloud)
- [Upjet framework](https://github.com/crossplane/upjet)
- [Argo CD automated sync and pruning](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)

References track upstream documentation. Select documentation matching the versions validated for this platform before implementing manifests.
