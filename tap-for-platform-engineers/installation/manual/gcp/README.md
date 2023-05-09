# Installation On Google Cloud Platform

## Environment Variables

In this section, we set up some environment variables that will be referenced down the line in the installation steps.

```bash
GCP_PROJECT_ID="..."
GCP_REGION="e.g. europe-west1"

GKE_CLUSTER_NAME="e.g. tap-cluster"

TANZUNET_USERNAME="..."
TANZUNET_PASSWORD="..."

TAP_DOMAIN="e.g. tap.example.com"
```


## Prerequisites

### GCP Preparations

Create the following GCP resources manually or use this [Terraform](https://github.com/unofficial-guide-to-tap/terraform/tree/main/gcp) project to automate the setup.

1. Create a **Virtual Private Cloud** (VPC)
2. Create a **Google Kubernetes Engine** (GKE) cluster in the VPC (check official docs for cluster requirements)
3. Create a **Cloud DNS** Managed Zone (or external DNS service)
4. Have access to **Google Container Registry** (GCR) (or external container registry)
5. A **Service Account Key** to be used by TAP 
6. An Ubuntu based **Google Compute Engine Instance** in the VPC that will be used as a jump host

<!--
END: ## Prepare The Infrastructure
-->

### Jump Host Setup

Before you proceed, install the following components:

| Software | Version | Check |
|---|---|---|
| Docker | >= 20.10 | `docker version` |
| Kubectl | >= 1.25  | `kubectl version --client` |
| gcloud | latest  | `gcloud version` |

#### Connect To Kubernetes

```bash
gcloud auth login
gcloud container clusters get-credentials $GKE_CLUSTER_NAME \
  --region $GCP_REGION \
  --project $GCP_PROJECT_ID
```

#### Service Account Key

Create a copy of the Service Account key at `$HOME/key.json`.
```bash
vim $HOME/key.json
```

### Download Software
Download the following artifacts from [Tanzu Network](https://network.tanzu.vmware.com/) to your jump host.

| Product | Release | Artifact | Notes |
|---|---|---|---|
| Cluster Essentials for VMware Tanzu | 1.5.0 | tanzu-cluster-essentials-linux-amd64-1.5.0.tgz | Contains [Carvel](https://carvel.dev/) tools |
| Cluster Essentials for VMware Tanzu | 1.5.0 | tanzu-cluster-essentials-bundle-1.5.0.yml | 1.5 | Contains the SHA hash |
| VMware Tanzu Application Platform | 1.5.0 | tanzu-cli-tap-1.5.0/tanzu-framework-bundle-linux | This is the name of the Tanzu CLI which is the primary interface for platform engineers and application teams to interact with TAP. The software bundle is part of the "Tanzu Application Platform" product in Tanzu Network |

## Installation

### Install Tanzu CLI

1. Extract the downloaded archive
    ```bash
    mkdir tanzu-framework
    tar xvf tanzu-framework-linux-amd64-v0.28.1.1.tar -C tanzu-framework
    cd tanzu-framework
    ```

2. Install the executable
    ```bash
    sudo install cli/core/v0.28.1/tanzu-core-linux_amd64 /usr/local/bin/tanzu
    ```

3. Install the shipped plugins
    ```bash
    export TANZU_CLI_NO_INIT=true
    tanzu plugin install --local cli all
    ```

4. Validate the installation
    ```bash
    tanzu version
    tanzu plugin list
    ```

### Install Carvel Tools
The binaries we need to have installed are shipping with the previously downloaded Cluster Essentials package.

1. Extract the downloaded archive

    ```bash
    mkdir cluster-essentials
    tar xvf tanzu-cluster-essentials-linux-amd64-1.5.0.tgz -C cluster-essentials
    cd cluster-essentials
    ```

2. Install the executables
    ```bash
    sudo install ./imgpkg /usr/local/bin/imgpkg
    sudo install ./kapp /usr/local/bin/kapp
    sudo install ./kbld /usr/local/bin/kbld
    sudo install ./ytt /usr/local/bin/ytt
    ```

3. Validate the installation
    ```bash
    imgpkg --version
    kapp --version
    kbld --version
    ytt --version
    ```

### Create A Package Repository Mirror

1. Docker login to Tanzu Network
    ```bash
    echo "$TANZUNET_PASSWORD" | \
    docker login registry.tanzu.vmware.com \
      -u $TANZUNET_USERNAME \
      --password-stdin
    ```

    If your receive a "permission denied" error in this step, your Linux user account probably just isn't in the `docker` group. Run the following command, then logout and and log back in again:

    ```bash
    sudo addgroup tapadmin docker
    ```

2. Docker login to Google Container Registry (GCR)
    ```bash
    gcloud auth login
    gcloud auth configure-docker
    ```

3. Mirror cluster essentials packages

    The SHA hash used in this section is taken from the file `foo` that you downloaded before. In case you're using a different version, you may extract that hash from the yaml with the following command:

    Get the SHA hash:
    ```bash
    CLUSTER_ESSENTIALS_SHA=$(cat tanzu-cluster-essentials-bundle-1.5.0.yml | yq '.bundle.image' | cut -d ":" -f 2)
    ```

    Run the mirror process:
    ```bash
    imgpkg copy \
      -b registry.tanzu.vmware.com/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:${CLUSTER_ESSENTIALS_SHA} \
      --to-repo gcr.io/${GCP_PROJECT_ID}/cluster-essentials-bundle \
      --include-non-distributable-layers
    ```

    The whole process should not take more than a minute or two.

3. Mirror TAP packages
    ```bash
    imgpkg copy \
      -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.5.0 \
      --to-repo gcr.io/${GCP_PROJECT_ID}/tap-packages \
      --include-non-distributable-layers
    ```

    The `tap-packages` are significantly larger than the `cluster-essentials`. This may easily take 10min or more.

<!--
END: ## Create A Package Repository Mirror
-->

### Install Cluster Essentials

1. Get the SHA hash

   ```bash
    CLUSTER_ESSENTIALS_SHA=$(cat tanzu-cluster-essentials-bundle-1.5.0.yml | yq '.bundle.image' | cut -d ":" -f 2)
    ```

2. Setup environment variables

    ```bash
    export INSTALL_REGISTRY_HOSTNAME="gcr.io"
    export INSTALL_BUNDLE="gcr.io/${GCP_PROJECT_ID}/cluster-essentials-bundle@sha256:$CLUSTER_ESSENTIALS_SHA"
    export INSTALL_REGISTRY_USERNAME="_json_key"
    export INSTALL_REGISTRY_PASSWORD="$(cat $HOME/key.json)"
    ```

3. Run the installation script
    ```bash
    cd cluster-essentials
    ./install.sh --yes
    ```

<!--
END: ## Install Cluster Essentials
-->

### Install The TAP Package

1. Create the installation `Namespace`
    ```bash
    kubectl create namespace tap-install
    ```

2. Create registry `Secret`

    ```bash
    tanzu secret registry add tap-registry \
      --username "_json_key" \
      --password-file $HOME/key.json \
      --server "gcr.io" \
      --export-to-all-namespaces --yes --namespace tap-install
    ```

2. Create `PackageRepository`

    ```bash
    tanzu package repository add tanzu-tap-repository \
      --namespace tap-install \
      --url gcr.io/${GCP_PROJECT_ID}/tap-packages:1.5.0
    ```

3. Create the TAP configuration file

    ```
    GOOGLE_ACCOUNT_KEY="$(cat $HOME/key.json | jq -c | jq -R | sed 's/^"//' | sed 's/"$//')"
    ```

    ```bash
    cat <<EOF > values.yaml
    shared:
      ingress_domain: $TAP_DOMAIN

      image_registry:
        project_path: "gcr.io/$GCP_PROJECT_ID"
        username: "_json_key"
        password: "$GOOGLE_ACCOUNT_KEY"

    ceip_policy_disclosed: true

    profile: full

    supply_chain: basic

    contour:
      envoy:
        service:
          type: LoadBalancer

    tap_gui:
      service_type: ClusterIP
    EOF
    ```

1. Install the `tap` package with that configuration

    ```bash
    tanzu package install tap \
      -p tap.tanzu.vmware.com \
      -v "1.5.0" \
      --values-file values.yaml \
      --wait="false" \
      -n "tap-install"
    ```

2. Watch progress of the `PackageInstall`s

    ```bash
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

### Create DNS Records

1. Get the public IP of your Contour ingress (installed with TAP)
    ```
    kubectl -n tanzu-system-ingress get svc envoy -o json \
      | jq ".status.loadBalancer.ingress[] | .ip" -r
    ```

2. Create the following A records pointing to that address
   - `*.$TAP_DOMAIN` 
   - `*.cnrs.$TAP_DOMAIN`

The [Terraform](https://github.com/unofficial-guide-to-tap/terraform/tree/main/gcp) project mentioned above allows you to configure the IP address as a variable so you can manage these entries via Terraform.

<!--
END: ## Create DNS Records
-->

## Validation <!-- omit from toc -->

### Access TAP GUI

1. Open your browser (ideally, use Google Chrome as it allows you to accept the self-signed certificate)
2. Navigate to [https://tap-gui.$TAP_DOMAIN](http://tap-gui.$TAP_DOMAIN)
3. In the "Guest" panel, click "ENTER" to proceed as a guest user

### Deploy A Test Workload

1. Create a developer namespace
    ```bash
    kubectl create ns --dry-run=client -o yaml test | kubectl apply -f -
    kubectl label namespaces test apps.tanzu.vmware.com/tap-ns=""
    ```

2. Deploy the workload
    ```bash
    tanzu apps workload create petclinic -n test \
      -l "app.kubernetes.io/part-of=petclinic" \
      -l "apps.tanzu.vmware.com/workload-type=web" \
      --build-env "BP_JVM_VERSION=17" \
      --git-repo https://github.com/spring-projects/spring-petclinic.git \
      --git-branch main
    ```

3. Monitor progress

    Run the following command and check the readiness and health state of supply chain resources:
    ```bash
    tanzu apps workload get petclinic -n test
    ```

    Tail the logs of the supply chain execution:
    ```bash
    tanzu apps workload tail petclinic --namespace test
    ```

    Watch the progress in the UI:
    1. Open your browser at [https://tap-gui.DOMAIN](http://tap-gui.DOMAIN)
    2. In the menu, navigate to "Supply Chains"
    3. Click on `petclinic`

    &nbsp;
    
4. Access the application

    1. Run the following command to retrieve the application URL. The URL is displayed in the "Knative Services" section.
        ```bash
        tanzu apps workload get petclinic --namespace test
        ```

    2. Open the URL in your browser. You should see the Petclinic web UI.

### Cleanup

1. Delete the `Workload`

    ```bash
    tanzu -n test app workload delete petclinic
    ```

2. Wait a few seconds until the resources got cleaned up

3. Delete the `Namespace`

    ```bash
    kubectl delete ns test
    ```
