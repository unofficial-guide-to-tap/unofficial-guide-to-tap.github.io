# Installation On Google Cloud Platform

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

[jumphost-software-table](../jumphost-software-table.md ':include')

#### Setup Parameters

In this section, we set up some environment variables that will be referenced down the line in the installation steps.

```bash
GCP_PROJECT_ID="my-awesome-project-1234"
GCP_REGION="europe-west1"
GKE_CLUSTER_NAME="tap-cluster"
TANZUNET_USERNAME="jdoe@vmware.com"
TANZUNET_PASSWORD="************"
TAP_DOMAIN="tap.example.com"
```

[params-file-instructions](../params-file-instructions.md ':include')

#### Login To GCP

```bash
gcloud auth login
```

#### Connect To Kubernetes

```bash
gcloud container clusters get-credentials $GKE_CLUSTER_NAME \
  --region $GCP_REGION \
  --project $GCP_PROJECT_ID
```

#### Service Account Key

Create a copy of the Service Account key at `$HOME/key.json`.
```bash
vim $HOME/key.json
```

#### Download Software
Download the following artifacts from [Tanzu Network](https://network.tanzu.vmware.com/) to your jump host.

[tanzunet-software-table](../tanzunet-software-table.md ':include')

## Installation

The steps in this section are designed to be executed on the jump host.

### Install Tanzu CLI

Start running the following commands from your home directory:

```
cd $HOME
```

[tanzucli-install-steps](../tanzucli-install-steps.md ':include')

### Install Carvel Tools
The binaries we need to have installed are shipping with the previously downloaded Cluster Essentials package.

Start running the following commands from your home directory:

```
cd $HOME
```

[carvel-install-steps](../carvel-install-steps.md ':include')

### Create A Package Repository Mirror

[docker-group-instructions](../docker-group-instructions.md ':include')

1. Docker login to Tanzu Network
    ```bash
    echo "$TANZUNET_PASSWORD" | \
    docker login registry.tanzu.vmware.com \
      -u $TANZUNET_USERNAME \
      --password-stdin
    ```

2. Docker login to Google Container Registry (GCR)
    ```bash
    gcloud auth configure-docker
    ```

3. Mirror cluster essentials packages

    The SHA hash used in this section is taken from the file `foo` that you downloaded before. In case you're using a different version, you may extract that hash from the yaml with the following command:

    Start running the following commands from your home directory:

    ```
    cd $HOME
    ```

    Get the SHA hash:
    ```bash
    CLUSTER_ESSENTIALS_SHA=$(cat $HOME/tanzu-cluster-essentials-bundle-1.5.0.yml | yq '.bundle.image' | cut -d ":" -f 2)
    ```

    Run the mirror process:
    ```bash
    imgpkg copy \
      -b registry.tanzu.vmware.com/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:${CLUSTER_ESSENTIALS_SHA} \
      --to-repo gcr.io/${GCP_PROJECT_ID}/cluster-essentials-bundle \
      --include-non-distributable-layers
    ```

    The whole process should not take more than a minute or two.

4. Mirror TAP packages
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

Start running the following commands from your home directory:

```
cd $HOME
```

1. Get the SHA hash if not already there

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

Start running the following commands from your home directory:

```
cd $HOME
```

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

    > **Note:** It is normal to see some `PackageInstall`s fail in the early stages. Give it time to reconcile.

    ```bash
    tanzu -n tap-install package installed list
    ```

<!--
END: ## Install TAP
-->

### Create DNS Records

[create-dns-records-steps](../create-dns-records-steps.md ':include')

The [Terraform](https://github.com/unofficial-guide-to-tap/terraform/tree/main/gcp) project mentioned above allows you to configure the IP address as a variable so you can manage these entries via Terraform.

<!--
END: ## Create DNS Records
-->

## Validation

[validation](../validation.md ':include')