1. Get the SHA hash if not already there

   ```bash
   CLUSTER_ESSENTIALS_SHA=$(cat tanzu-cluster-essentials-bundle-1.5.0.yml | yq '.bundle.image' | cut -d ":" -f 2)
   ```

2. Setup environment variables

    ```bash
    export INSTALL_REGISTRY_HOSTNAME="${AZR_REGISTRY_HOST}"
    export INSTALL_BUNDLE="${AZR_REGISTRY_HOST}/cluster-essentials-bundle@sha256:$CLUSTER_ESSENTIALS_SHA"
    export INSTALL_REGISTRY_USERNAME="${AZR_REGISTRY_USERNAME}"
    export INSTALL_REGISTRY_PASSWORD="${AZR_REGISTRY_PASSWORD}"
    ```

3. Run the installation script
    ```bash
    cd cluster-essentials
    ./install.sh --yes
    ```