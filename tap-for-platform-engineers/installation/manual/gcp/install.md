# TAP on GCP - Installation

- [Install Tanzu CLI](#install-tanzu-cli)
- [Install Carvel Tools](#install-carvel-tools)
- [Create A Package Repository Mirror](#create-a-package-repository-mirror)
- [Install Cluster Essentials](#install-cluster-essentials)
- [Install The TAP Package](#install-the-tap-package)
- [Create DNS Records](#create-dns-records)

## Install Tanzu CLI

1. Extract the downloaded archive
    ```
    mkdir tanzu-framework
    tar xvf tanzu-framework-linux-amd64-v0.25.4.6.tar -C tanzu-framework
    cd tanzu-framework
    ```

2. Install the executable
    ```
    sudo install cli/core/v0.25.4/tanzu-core-linux_amd64 /usr/local/bin/tanzu
    ```

3. Install the shipped plugins
    ```
    export TANZU_CLI_NO_INIT=true
    tanzu plugin install --local cli all
    ```

4. Validate the installation
    ```
    tanzu version
    tanzu plugin list
    ```

## Install Carvel Tools
The binaries we need to have installed are shipping with the previously downloaded Cluster Essentials package.

1. Extract the downloaded archive

    ```
    mkdir cluster-essentials
    tar xvf tanzu-cluster-essentials-linux-amd64-1.4.1.tgz -C cluster-essentials
    cd cluster-essentials
    ```

2. Install the executables
    ```
    sudo install ./imgpkg /usr/local/bin/imgpkg
    sudo install ./kapp /usr/local/bin/kapp
    sudo install ./kbld /usr/local/bin/kbld
    sudo install ./ytt /usr/local/bin/ytt
    ```

3. Validate the installation
    ```
    imgpkg --version
    kapp --version
    kbld --version
    ytt --version
    ```

## Create A Package Repository Mirror

1. Docker login to Tanzu Network
    ```
    TANZUNET_USERNAME="..."
    TANZUNET_PASSWORD="..."

    echo "$TANZUNET_PASSWORD" | \
    docker login registry.tanzu.vmware.com \
        -u $TANZUNET_USERNAME \
        --password-stdin
    ```

2. Docker login to Google Container Registry (GCR)
    ```
    gcloud auth login
    gcloud auth configure-docker
    ```

3. Mirror cluster essentials packages

    ```
    SHA="2354688e46d4bb4060f74fca069513c9b42ffa17a0a6d5b0dbb81ed52242ea44"
    HOST="gcr.io"
    REPO="..."

    imgpkg copy \
    -b registry.tanzu.vmware.com/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:${SHA} \
    --to-repo ${HOST}/${REPO}/cluster-essentials-bundle \
    --include-non-distributable-layers
    ```

3. Mirror TAP packages
    ```
    VERSION="1.4.4"
    HOST="gcr.io"
    REPO="..."

    imgpkg copy \
    -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:${VERSION} \
    --to-repo ${HOST}/${REPO}/tap-packages \
    --include-non-distributable-layers
    ```

<!--
END: ## Create A Package Repository Mirror
-->

## Install Cluster Essentials

1. Setup environment variables
    ```
    GOOGLE_APPLICATION_CREDENTIALS="$HOME/key.json"
    SHA="2354688e46d4bb4060f74fca069513c9b42ffa17a0a6d5b0dbb81ed52242ea44"
    HOST="gcr.io"
    REPO="..."

    export INSTALL_REGISTRY_HOSTNAME="$HOST"
    export INSTALL_BUNDLE="$HOST/$REPO/cluster-essentials-bundle@sha256:$SHA"
    export INSTALL_REGISTRY_USERNAME="_json_key"
    export INSTALL_REGISTRY_PASSWORD="$(cat $GOOGLE_APPLICATION_CREDENTIALS)"
    ```

2. Run the installation script
    ```
    cd cluster-essentials
    ./install.sh --yes
    ```

<!--
END: ## Install Cluster Essentials
-->

## Install The TAP Package

### Prepare The Installation Namespace

1. Create the installation `Namespace`
    ```
    kubectl create namespace tap-install
    ```

2. Create registry `Secret`

    ```
    tanzu secret registry add tap-registry \
    --username "_json_key" \
    --password-file $HOME/key.json \
    --server "gcr.io" \
    --export-to-all-namespaces --yes --namespace tap-install
    ```

2. Create `PackageRepository`

    ```
    VERSION="1.4.4"
    HOST="gcr.io"
    REPO="..."

    tanzu package repository add tanzu-tap-repository \
    --namespace tap-install \
    --url ${HOST}/${REPO}/tap-packages:${VERSION}
    ```

3. Create the TAP configuration file

    ```
    vim values.yaml
    ```
    ```
    shared:
      ingress_domain: YOUR_DOMAIN

      image_registry:
        project_path: "REGISTRY_HOST/REGISTRY_REPO"
        username: "_json_key"
        password: |
          SERVICE_ACCOUNT_KEY_JSON

    ceip_policy_disclosed: true

    profile: full

    supply_chain: basic

    contour:
      envoy:
        service:
          type: LoadBalancer

    tap_gui:
      service_type: ClusterIP

    cnrs:
      domain_template: "{{.Name}}-{{.Namespace}}.{{.Domain}}"
    ```

4. Install the `tap` package with that configuration

    ```
    tanzu package install tap \
        -p tap.tanzu.vmware.com \
        -v "1.4.4" \
        --values-file values.yaml \
        --wait="false" \
        -n "tap-install"
    ```

3. Watch progress of the `PackageInstall`s

    ```
    tanzu -n tap-install packages installed list
    ```

<!--
END: ## Install TAP
-->

## Create DNS Records

1. Get the public IP of your Contour ingress (installed with TAP)
    ```
    kubectl -n tanzu-system-ingress \
    get svc envoy \
    -o json \
    | jq ".status.loadBalancer.ingress[] | .ip" -r
    ```

2. Create the following A records pointing to that address
   - `*.DOMAIN` 
   - `*.cnrs.DOMAIN`

The [Terraform](https://github.com/unofficial-guide-to-tap/terraform/tree/main/gcp) project mentioned above allows you to configure the IP address as a variable so you can manage these entries via Terraform.

<!--
END: ## Create DNS Records
-->

---
Next: [Validation](./validate.md)