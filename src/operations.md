## Upgrade

### Flux

This is automated using [Github Action](https://github.com/buvis/clusters/blob/main/.github/workflows/update-flux-home.yaml).

### Nodes

TODO: I switched to Talos, so I have to figure this out

## Recreate

### Backup

1. Run backup job manually for every PVC: `buvisctl backup -n <NAMESPACE> <PVC>`
2. Perform application specific backup for the applications considered critical
    - home-assistant
    - linkace
    - monica

### Destroy

Run `buvisctl destroy` in cluster's directory.

### Bootstrap

1. Set workstation's DNS to 1.1.1.1, because Blocky isn't running
2. Run `buvisctl bootstrap` in cluster's directory.

### Restore

Repeat for every PVC: `buvisctl backup -n <NAMESPACE> <PVC>`


## Usage

### Use secret values in manifests directly
1. Define key-value pair in [cluster-secrets](https://github.com/buvis/clusters/blob/main/production/operations/flux-system/extras/cluster-secrets.yaml)
    ```bash
    sops flux-system/extras/cluster-secrets.yaml
    ```
2. Refer to the secret value in manifest file
    ``` yaml
    password: ${SECRET_PASSWORD}
    ```

### Use secret values in manifests from a file
1. Create `secret.yaml`
    ```yaml
    ---
    apiVersion: v1
    kind: Secret
    metadata:
        name: oauth2-proxy-secrets
    type: Opaque
    stringData:
        clientID: "<secret_value>"
        clientSecret: "<secret_value>"
    ```
2. Include secrets into manifest
    ``` yaml
    clientID:
    valueFrom:
        secretKeyRef:
        name: oauth2-proxy-secrets
        key: clientID
    ```
3. Encrypt the secrets before pushing (TODO: use pre-commit hook)
    ```bash
    sops --encrypt --in-place secret.yaml
    ```

### Wait for another helm release (dependency)
Some releases depend on others. You may get errors when reconciling such releases too early.

Flux will reconcile something only once something else it depends on is available when **dependsOn** is mentionned in helm release manifest. It can refer to a name of another helm release across all namespaces.

See [Flux documentation](https://fluxcd.io/docs/components/helm/helmreleases/)

#### Example

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
name: longhorn
namespace: longhorn-system
spec:
dependsOn:
- name: ingress-nginx
    namespace: kube-system
- name: oauth2-proxy
    namespace: kube-system
- name: cert-manager
    namespace: kube-system
```

## Troubleshooting

### Flux can't reconcile a helmrelease
- Get status of all helmreleases
    ```bash
    flux get helmreleases --all-namespaces
    ```
    or shorter
    ```bash
    flux get hr -A
    ```
- Make sure that sources can be accessed (as there might be typos)
    ```bash
    flux get sources helm -A
    flux get sources git -A
    flux get sources chart -A
    ```
- Get more information on helm release (start here when "install retries exhausted")
    ```bash
    kubectl describe helmrelease <RELEASE_NAME> -n <RELEASE_NAMESPACE>
    ```
- Restart when Flux gave up
    ```bash
    flux resume hr <RELEASE_NAME> -n <RELEASE_NAMESPACE>
    ```

### Get CPU temperature in Proxmox for Asus PN50

Look at k10temp-pci-00c3 in the output of `sensors` command.
