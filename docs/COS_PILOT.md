# Development COS bucket pilot

This repository declares three auto-sync Argo CD Applications: the Tencent
Cloud Provider (`v0.8.6`), ProviderConfig and a private development COS Bucket.
The foundation's root Applications automatically synchronize the AppProject
and child Application definitions. Each child automatically syncs its own resources
and repairs Kubernetes manifest drift (`selfHeal: true`). `cos-dev` enables
automatic pruning and allows an empty resource set. Its Bucket uses
`deletionPolicy: Delete` and `forProvider.forceClean: true` to empty this test
bucket before deleting it. Provider and ProviderConfig pruning remains disabled;
no cascading Application deletion finalizers are configured. Merging changes
into `main` can trigger cloud resource creation, updates or deletion.

Failed syncs retry up to 10 times with exponential backoff (10 seconds initially,
capped at 3 minutes). These retries help with temporarily missing CRDs but do not
establish Provider or credential readiness. After retries are exhausted, resolve
the cause and retry the affected child Application.

## Before deployment

1. Confirm `spec.forProvider.bucket` in `environments/dev/cos/bucket.yaml` ends
   with your Tencent Cloud APPID, not SecretId. Use a new, unused name outside existing Terraform
   ownership. The full cloud bucket name must end with `-<APPID>`.
2. Review/apply the foundation's Terraform credential change through its normal
   plan and approval process. It creates `crossplane-system/tencentcloud-credentials`
   using the existing sensitive `secret_id` and `secret_key` inputs. Its
   `credentials` key contains JSON with `secret_id`, `secret_key` and `region`.
   The foundation's development configuration sets `crossplane_region` to
   `ap-guangzhou` without changing the cluster's region.
3. If the Secret was previously created by a script, import it into Terraform
   once after review; see `modules/k8s_cicd/CROSSPLANE.md` in `tencentcloud-k8s`.
   Do not also manage the Secret with Argo CD or a script. Protect Terraform state
   and saved plans, which contain credentials. Review the cloud identity's
   permissions; prefer permissions limited to the intended test operations.
4. Render the child Application entry point locally:

   ```sh
   kubectl kustomize argocd/applications
   ```

   Expect three Applications with automated sync, `selfHeal: true` and bounded
   retries. Only `cos-dev` has `prune: true` and `allowEmpty: true`; the platform
   Applications keep both false. The ProviderConfig and Bucket declarations
   are raw YAML in separate source directories; Argo CD renders them as
   directory sources.

## Deployment and verification

1. Review and merge these files into `main`. Wait for `infra-gitops-projects`
   and `infra-gitops-applications` to synchronize. The `infrastructure` project
   must permit the ProviderConfig and Bucket kinds before syncing them.
2. The `tencentcloud-provider` child automatically syncs. Wait for the Provider
   object to appear, then check installation, package health and both CRDs:

   ```sh
   kubectl wait --for=condition=Installed provider.pkg.crossplane.io/provider-tencentcloud --timeout=300s
   kubectl wait --for=condition=Healthy provider.pkg.crossplane.io/provider-tencentcloud --timeout=300s
   kubectl wait --for=condition=Established crd/providerconfigs.tencentcloud.crossplane.io crd/buckets.cos.tencentcloud.crossplane.io --timeout=120s
   ```

   The pinned version is a test candidate. Schema validation does not establish
   runtime compatibility with Crossplane `1.20.3` and Kubernetes `1.28.8`. Resolve
   package health errors before continuing. Child Application sync waves alone
   do not establish Provider/CRD readiness.
3. Check the Terraform-managed Secret exists without printing its contents:

   ```sh
   kubectl -n crossplane-system get secret tencentcloud-credentials
   ```

4. Verify the automatically synchronized ProviderConfig exists:

   ```sh
   kubectl get providerconfigs.tencentcloud.crossplane.io tencentcloud-dev
   ```

   ProviderConfig `tencentcloud-dev` only
   references that Secret; it contains no keys. Its health does not itself
   establish that Tencent Cloud authentication works. If the foundation uses
   a custom Crossplane namespace, update the Secret reference and AppProject
   destination accordingly.
5. Review the bucket name and changes before merging. The `cos-dev` child
   automatically syncs; this creates a real private COS bucket and may incur
   cloud charges. Verify the resulting resource:

   ```sh
   kubectl get buckets.cos.tencentcloud.crossplane.io
   kubectl describe bucket.cos.tencentcloud.crossplane.io crossplane-cos-test
   kubectl wait --for=condition=Ready bucket.cos.tencentcloud.crossplane.io/crossplane-cos-test --timeout=300s
   kubectl wait --for=condition=Synced bucket.cos.tencentcloud.crossplane.io/crossplane-cos-test --timeout=300s
   ```

   Verify the full name, Guangzhou region and private access in Tencent Cloud.
   For failures, inspect resource conditions, Provider pod logs and cluster
   events without exposing credentials.

## Cleanup

First merge and synchronize the deletion-policy changes while the bucket
manifest still exists. Verify the live policies before removing its declaration:

```sh
kubectl -n argocd get application cos-dev -o jsonpath='{.spec.syncPolicy.automated}{"\n"}'
kubectl get buckets.cos.tencentcloud.crossplane.io crossplane-cos-test -o jsonpath='{.spec.deletionPolicy}{" | forceClean="}{.spec.forProvider.forceClean}{"\n"}'
```

Expect `prune: true`, `allowEmpty: true`, Bucket deletion policy `Delete` and
`forceClean: true`.
Then remove `environments/dev/cos/bucket.yaml` and merge into `main`. Argo CD
prunes the Kubernetes Bucket object; Crossplane deletes the corresponding cloud
bucket. Keep `environments/dev/cos/.gitkeep`, the `cos-dev` Application, Provider,
ProviderConfig and credentials until deletion completes. The placeholder keeps
the source directory present when the last manifest is removed; `allowEmpty`
does not bypass a missing-path/render error. Removing only the Application YAML
will not trigger this cleanup because parent pruning is disabled.

Deletion is asynchronous and requires cloud permissions to list and delete
objects as well as delete the bucket. `forceClean: true` removes objects before
bucket deletion, so test data will be lost. It does not bypass object retention
or missing permissions. If deletion is blocked, inspect the Bucket events and
Provider logs and resolve the reported cause. Verify both the Kubernetes object and
the cloud bucket are gone; do not remove finalizers to bypass failed deletion.
Do not delete the cloud bucket directly while its Git declaration is active,
because the Provider may recreate it.

Do not uninstall the Provider or Crossplane before handling managed-resource
objects and recording their external identities. This pilot does not adopt
existing Terraform resources or implement automatic readiness gates.

## Pinned upstream schemas

- [ProviderConfig](https://github.com/crossplane-contrib/provider-tencentcloud/blob/v0.8.6/package/crds/tencentcloud.crossplane.io_providerconfigs.yaml)
- [COS Bucket](https://github.com/crossplane-contrib/provider-tencentcloud/blob/v0.8.6/package/crds/cos.tencentcloud.crossplane.io_buckets.yaml)
- [Credential fields](https://github.com/crossplane-contrib/provider-tencentcloud/blob/v0.8.6/internal/clients/tencentcloud.go)
