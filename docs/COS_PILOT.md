# Development COS bucket pilot

This repository declares three manual-sync Argo CD Applications: the Tencent
Cloud Provider (`v0.8.6`), ProviderConfig and a private development COS Bucket.
The foundation's root Applications automatically synchronize the AppProject
and child Application definitions, but do not sync the children themselves.

## Before deployment

1. Replace `REPLACE_APPID` in `environments/dev/cos/bucket.yaml` with your Tencent
   Cloud APPID, not SecretId. Use a new, unused name outside existing Terraform
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

   Expect three Applications, each with `syncPolicy: {}` and no cascading
   deletion finalizer. The ProviderConfig and Bucket declarations are raw YAML
   in separate source directories; Argo CD renders them as directory sources.

## Manual deployment order

1. Review and merge these files into `main`. Wait for `infra-gitops-projects`
   and `infra-gitops-applications` to synchronize. The `infrastructure` project
   must permit the ProviderConfig and Bucket kinds before syncing them.
2. In Argo CD, manually sync `tencentcloud-provider`, then wait for installation,
   package health and both CRDs:

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

4. Manually sync `tencentcloud-config`. ProviderConfig `tencentcloud-dev` only
   references that Secret; it contains no keys. Its health does not itself
   establish that Tencent Cloud authentication works. If the foundation uses
   a custom Crossplane namespace, update the Secret reference and AppProject
   destination accordingly.
5. Confirm the bucket APPID placeholder is replaced and merged. Review the Argo
   CD diff for `cos-dev`, then manually sync it. This creates a real private
   COS bucket and may incur cloud charges.

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

`deletionPolicy: Orphan` retains the cloud bucket when its Kubernetes declaration
is deleted. Automatic pruning is disabled. After review, remove the Git
declaration and explicitly delete its Kubernetes object, then separately clean
up the retained cloud bucket. Do not delete the cloud bucket while its active
declaration remains reconciled; the Provider may recreate it. Check objects,
versions and retention before declaring cleanup complete.

Do not uninstall the Provider or Crossplane before handling managed-resource
objects and recording their external identities. This pilot does not adopt
existing Terraform resources or implement automatic readiness gates.

## Pinned upstream schemas

- [ProviderConfig](https://github.com/crossplane-contrib/provider-tencentcloud/blob/v0.8.6/package/crds/tencentcloud.crossplane.io_providerconfigs.yaml)
- [COS Bucket](https://github.com/crossplane-contrib/provider-tencentcloud/blob/v0.8.6/package/crds/cos.tencentcloud.crossplane.io_buckets.yaml)
- [Credential fields](https://github.com/crossplane-contrib/provider-tencentcloud/blob/v0.8.6/internal/clients/tencentcloud.go)
