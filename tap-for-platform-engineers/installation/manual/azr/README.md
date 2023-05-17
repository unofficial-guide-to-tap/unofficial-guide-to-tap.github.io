# Installation On Microsoft Azure

## Prerequisites

### Azure Preparations

Create the following Azure resources manually or use this [Terraform](https://github.com/unofficial-guide-to-tap/terraform/tree/main/azr) project to automate the setup.

1. Create a **Virtual Network** and **Subnet**
2. Create a **Azure Kubernetes Service** (AKS) cluster in the network
3. Create a **DNS Zone** (or external DNS service)
4. Create a **Azure Container Registry** (ACR) with a token and token password that has access to the repositories `tap-packages`, `cluster-essentials-bundle`, `buildservice` and `workloads`
5. An Ubuntu based **Virtual Machine** in the subnet that will be used as a jump host

<!--
END: ## Prepare The Infrastructure
-->

### Jump Host Setup

Before you proceed, install the following components:

[jumphost-software-table](../jumphost-software-table.md ':include')

#### Setup Parameters

In this section, we set up some environment variables that will be referenced down the line in the installation steps.

```bash
AZR_SUBSCRIPTION_ID="?"
AZR_RESOURCE_GROUP="?"

AZR_REGISTRY_HOST="?"
AZR_REGISTRY_USERNAME="?"
AZR_REGISTRY_PASSWORD="?"

TANZUNET_USERNAME="jdoe@vmware.com"
TANZUNET_PASSWORD="************"
TAP_DOMAIN="tap.example.com"
```

[params-file-instructions](../params-file-instructions.md ':include')

#### Login To Azure

```bash
az login
```

#### Connect To Kubernetes

```bash
az account set --subscription $AZR_SUBSCRIPTION_ID
az aks get-credentials --resource-group $AZR_RESOURCE_GROUP --name tap-demo-cluster
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

2. Docker login to Azure Container Registry (ACR)
    ```bash
    echo "$AZR_REGISTRY_PASSWORD" | \
    docker login "$AZR_REGISTRY_HOST" \
      -u $AZR_REGISTRY_USERNAME \
      --password-stdin
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
      --to-repo ${AZR_REGISTRY_HOST}/cluster-essentials-bundle \
      --include-non-distributable-layers
    ```

    The whole process should not take more than a minute or two.

4. Mirror TAP packages
    ```bash
    imgpkg copy \
      -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.5.0 \
      --to-repo ${AZR_REGISTRY_HOST}/tap-packages \
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

[cluster-essentials-install-steps](../cluster-essentials-install-steps.md ':include')

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
      --username "${AZR_REGISTRY_USERNAME}" \
      --password "${AZR_REGISTRY_PASSWORD}" \
      --server "${AZR_REGISTRY_HOST}" \
      --export-to-all-namespaces --yes --namespace tap-install
    ```

3. Create `PackageRepository`

    ```bash
    tanzu package repository add tanzu-tap-repository \
      --namespace tap-install \
      --url ${AZR_REGISTRY_HOST}/tap-packages:1.5.0
    ```

4. Create the TAP configuration file

    ```bash
    cat <<EOF > values.yaml
    shared:
      ingress_domain: $TAP_DOMAIN

      image_registry:
        project_path: "$AZR_REGISTRY_HOST/"
        username: "$AZR_REGISTRY_USERNAME"
        password: "$AZR_REGISTRY_PASSWORD"

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

5. Install the `tap` package with that configuration

    ```bash
    tanzu package install tap \
      -p tap.tanzu.vmware.com \
      -v "1.5.0" \
      --values-file values.yaml \
      --wait="false" \
      -n "tap-install"
    ```

6. Watch progress of the `PackageInstall`s

    > **Note:** It is normal to see some `PackageInstall`s fail in the early stages. Give it time to reconcile.

    ```bash
    tanzu -n tap-install package installed list
    ```

<!--
END: ## Install TAP
-->

### Create DNS Records

[create-dns-records-steps](../create-dns-records-steps.md ':include')

<!--
END: ## Create DNS Records
-->

## Validation <!-- omit from toc -->

[validation](../validation.md ':include')