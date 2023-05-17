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

| Software | Version | Check |
|---|---|---|
| Docker | >= 20.10 | `docker version` |
| Kubectl | >= 1.25  | `kubectl version --client` |
| Azure CLI | latest  | `az version` |

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

You may e.g. copy the code above, edit and execute it in your shell using `EDITOR=vim fc`. Alternatively, save the copy above to a file like `params.sh` and load it into your shell with `source params.sh`. The latter makes it easier to load them again after you have exited your shell.

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

| Product | Release | Artifact | Notes |
|---|---|---|---|
| Cluster Essentials for VMware Tanzu | 1.5.0 | tanzu-cluster-essentials-linux-amd64-1.5.0.tgz | Contains [Carvel](https://carvel.dev/) tools |
| Cluster Essentials for VMware Tanzu | 1.5.0 | tanzu-cluster-essentials-bundle-1.5.0.yml | Contains the SHA hash |
| VMware Tanzu Application Platform | 1.5.0 | tanzu-cli-tap-1.5.0/tanzu-framework-bundle-linux | This is the name of the Tanzu CLI which is the primary interface for platform engineers and application teams to interact with TAP. The software bundle is part of the "Tanzu Application Platform" product in Tanzu Network |

## Installation

The steps in this section are designed to be executed on the jump host.

### Install Tanzu CLI

Start running the following commands from your home directory:

```
cd $HOME
```

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

Start running the following commands from your home directory:

```
cd $HOME
```

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

    ```
    GOOGLE_ACCOUNT_KEY="$(cat $HOME/key.json | jq -c | jq -R | sed 's/^"//' | sed 's/"$//')"
    ```

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
