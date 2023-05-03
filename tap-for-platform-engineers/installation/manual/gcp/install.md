# TAP on GCP - Installation <!-- omit from toc -->

- [Install Tanzu CLI](#install-tanzu-cli)
- [Install Carvel Tools](#install-carvel-tools)
- [Create A Package Repository Mirror](#create-a-package-repository-mirror)
- [Install Cluster Essentials](#install-cluster-essentials)
- [Install The TAP Package](#install-the-tap-package)
- [Create DNS Records](#create-dns-records)

---

## Install Tanzu CLI

1. Extract the downloaded archive
    ```
    mkdir tanzu-framework
    tar xvf tanzu-framework-linux-amd64-v0.28.1.1.tar -C tanzu-framework
    cd tanzu-framework
    ```

2. Install the executable
    ```
    sudo install cli/core/v0.28.1/tanzu-core-linux_amd64 /usr/local/bin/tanzu
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
    tar xvf tanzu-cluster-essentials-linux-amd64-1.5.0.tgz -C cluster-essentials
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

    If your receive a "permission denied" error in this step, your Linux user account probably just isn't in the `docker` group. Run the following command, then logout and and log back in again:

    ```
    sudo addgroup tapadmin docker
    ```

2. Docker login to Google Container Registry (GCR)
    ```
    # gcloud auth login # If you haven't already done so
    gcloud auth configure-docker
    ```

3. Mirror cluster essentials packages

    The SHA hash used in this section is taken from the file `foo` that you downloaded before. In case you're using a different version, you may extract that hash from the yaml with the following command:

    ```
    cat tanzu-cluster-essentials-bundle-1.5.0.yml | yq '.bundle.image' | cut -d ":" -f 2
    ```

    ```
    SHA="79abddbc3b49b44fc368fede0dab93c266ff7c1fe305e2d555ed52d00361b446"
    HOST="gcr.io"
    REPO="YOUR_PROJECT_ID"

    imgpkg copy \
      -b registry.tanzu.vmware.com/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:${SHA} \
      --to-repo ${HOST}/${REPO}/cluster-essentials-bundle \
      --include-non-distributable-layers
    ```

    The whole process should not take more than a minute or two.

3. Mirror TAP packages
    ```
    VERSION="1.5.0"
    HOST="gcr.io"
    REPO="YOUR_PROJECT_ID"

    imgpkg copy \
      -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:${VERSION} \
      --to-repo ${HOST}/${REPO}/tap-packages \
      --include-non-distributable-layers
    ```

    The `tap-packages` are significantly larger than the `cluster-essentials`. This may easily take 10min or more.

<!--
END: ## Create A Package Repository Mirror
-->

## Install Cluster Essentials

1. Setup environment variables
    ```
    GOOGLE_APPLICATION_CREDENTIALS="$HOME/key.json"
    SHA="79abddbc3b49b44fc368fede0dab93c266ff7c1fe305e2d555ed52d00361b446"
    HOST="gcr.io"
    REPO="YOUR_PROJECT_ID"

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
    VERSION="1.5.0"
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

    # TODO: We should keep the default here for a minimum installation and make
    # that change when configuring SSL/TLS depending on the method we use
    cnrs:
      domain_template: "{{.Name}}-{{.Namespace}}.{{.Domain}}"
    ```

1. Install the `tap` package with that configuration

    ```
    tanzu package install tap \
      -p tap.tanzu.vmware.com \
      -v "1.5.0" \
      --values-file values.yaml \
      --wait="false" \
      -n "tap-install"
    ```

2. Watch progress of the `PackageInstall`s

    ```
    tanzu -n tap-install package installed list
    ```

    It will take a good while depending on your environment before all `Packages` have been successfuly reconciled. Expected output:
    ```
    NAME                      PACKAGE-NAME                                 PACKAGE-VERSION   STATUS
    accelerator               accelerator.apps.tanzu.vmware.com            1.5.1             Reconcile succeeded
    api-auto-registration     apis.apps.tanzu.vmware.com                   0.3.0             Reconcile succeeded
    api-portal                api-portal.tanzu.vmware.com                  1.3.0             Reconcile succeeded
    appliveview               backend.appliveview.tanzu.vmware.com         1.5.1             Reconcile succeeded
    appliveview-apiserver     apiserver.appliveview.tanzu.vmware.com       1.5.1             Reconcile succeeded
    appliveview-connector     connector.appliveview.tanzu.vmware.com       1.5.1             Reconcile succeeded
    appliveview-conventions   conventions.appliveview.tanzu.vmware.com     1.5.1             Reconcile succeeded
    appsso                    sso.apps.tanzu.vmware.com                    3.1.0             Reconcile succeeded
    bitnami-services          bitnami.services.tanzu.vmware.com            0.1.0             Reconcile succeeded
    buildservice              buildservice.tanzu.vmware.com                1.10.8            Reconcile succeeded
    cartographer              cartographer.tanzu.vmware.com                0.7.1+tap.1       Reconcile succeeded
    cert-manager              cert-manager.tanzu.vmware.com                2.3.0             Reconcile succeeded
    cnrs                      cnrs.tanzu.vmware.com                        2.2.0             Reconcile succeeded
    contour                   contour.tanzu.vmware.com                     1.22.5+tap.1.5.0  Reconcile succeeded
    crossplane                crossplane.tanzu.vmware.com                  0.1.1             Reconcile succeeded
    developer-conventions     developer-conventions.tanzu.vmware.com       0.10.0            Reconcile succeeded
    eventing                  eventing.tanzu.vmware.com                    2.2.1             Reconcile succeeded
    fluxcd-source-controller  fluxcd.source.controller.tanzu.vmware.com    0.27.0+tap.10     Reconcile succeeded
    grype                     grype.scanning.apps.tanzu.vmware.com         1.5.0             Reconcile succeeded
    learningcenter            learningcenter.tanzu.vmware.com              0.2.7             Reconcile succeeded
    learningcenter-workshops  workshops.learningcenter.tanzu.vmware.com    0.2.6             Reconcile succeeded
    metadata-store            metadata-store.apps.tanzu.vmware.com         1.5.0             Reconcile succeeded
    namespace-provisioner     namespace-provisioner.apps.tanzu.vmware.com  0.3.1             Reconcile succeeded
    ootb-delivery-basic       ootb-delivery-basic.tanzu.vmware.com         0.12.5            Reconcile succeeded
    ootb-supply-chain-basic   ootb-supply-chain-basic.tanzu.vmware.com     0.12.5            Reconcile succeeded
    ootb-templates            ootb-templates.tanzu.vmware.com              0.12.5            Reconcile succeeded
    policy-controller         policy.apps.tanzu.vmware.com                 1.4.0             Reconcile succeeded
    scanning                  scanning.apps.tanzu.vmware.com               1.5.2             Reconcile succeeded
    service-bindings          service-bindings.labs.vmware.com             0.9.1             Reconcile succeeded
    services-toolkit          services-toolkit.tanzu.vmware.com            0.10.1            Reconcile succeeded
    source-controller         controller.source.apps.tanzu.vmware.com      0.7.0             Reconcile succeeded
    spring-boot-conventions   spring-boot-conventions.tanzu.vmware.com     1.5.1             Reconcile succeeded
    tap                       tap.tanzu.vmware.com                         1.5.0             Reconcile succeeded
    tap-auth                  tap-auth.tanzu.vmware.com                    1.1.0             Reconcile succeeded
    tap-gui                   tap-gui.tanzu.vmware.com                     1.5.1             Reconcile succeeded
    tap-telemetry             tap-telemetry.tanzu.vmware.com               0.5.0-build.3     Reconcile succeeded
    tekton-pipelines          tekton.tanzu.vmware.com                      0.41.0+tap.8      Reconcile succeeded
    ```

<!--
END: ## Install TAP
-->

## Create DNS Records

1. Get the public IP of your Contour ingress (installed with TAP)
    ```
    kubectl -n tanzu-system-ingress get svc envoy -o json \
      | jq ".status.loadBalancer.ingress[] | .ip" -r
    ```

2. Create the following A records pointing to that address
   - `*.DOMAIN` 
   - `*.cnrs.DOMAIN`

The [Terraform](https://github.com/unofficial-guide-to-tap/terraform/tree/main/gcp) project mentioned above allows you to configure the IP address as a variable so you can manage these entries via Terraform.

<!--
END: ## Create DNS Records
-->

</br>

---
Next: [Validation](./validate.md)
