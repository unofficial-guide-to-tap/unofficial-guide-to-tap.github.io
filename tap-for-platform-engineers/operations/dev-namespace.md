As a **Platform Engineer**, I want to
# Create A Developer Namespace

The functionality of TAP cannot be use from within any `Namespace`. A "TAP-enabled" `Namespace` is commonly called **developer namespace** and includes some additional configuration.

Instead of creating all necessary configuration manually, this is usually automated via the Namespace Provisioner which gets installed as part of any TAP deployment. To trigger the Namespace Provisioner, the admin needs to 

1. Create the `Namespace`

    ```
    kubectl create ns --dry-run=client -o yaml test | kubectl apply -f -
    ```

2. Label the `Namespace` the right key and value

    ```
    kubectl label namespaces test apps.tanzu.vmware.com/tap-ns=""
    ```

The exact resources deployed to a developer namespace, depend mainly on the cluster profile (view, iterate, build, run) the cluster is running. In a single cluster setup, all resources are created. To view the YTT that gets rendered into the developer namespace run the following command:

```
kubectl -n tap-namespace-provisioning get secret default-resources \
  -o json | jq '.data."content.yaml"' -r | base64 -d
```