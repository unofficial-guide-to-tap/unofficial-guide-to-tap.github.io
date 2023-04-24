TAP On GCP
=

<!-- TOC -->

- [TAP On GCP](#tap-on-gcp)
    - [Prepare The Infrastructure](#prepare-the-infrastructure)
    - [Download Software](#download-software)
    - [Prepare The Jump Host](#prepare-the-jump-host)
        - [Install Tanzu CLI](#install-tanzu-cli)
        - [Install Carvel Tools](#install-carvel-tools)
    - [Create A Package Repository Mirror](#create-a-package-repository-mirror)
    - [Install Cluster Essentials](#install-cluster-essentials)
    - [Install TAP](#install-tap)
    - [Create DNS Records](#create-dns-records)
    - [Validate The Installation](#validate-the-installation)
        - [Access TAP GUI](#access-tap-gui)
        - [Deploy A Test Workload](#deploy-a-test-workload)

<!-- /TOC -->


## Prepare The Infrastructure

Create the following GCP resources manually or use this [Terraform](https://github.com/unofficial-guide-to-tap/terraform/tree/main/gcp) project to automate the setup.

1. Create a **Virtual Private Cloud** (VPC)
2. Create a **Google Kubernetes Engine** (GKE) cluster in the VPC (check official docs for cluster requirements)
3. Create a **Cloud DNS** Managed Zone (or external DNS service)
4. Have access to **Google Container Registry** (GCR) (or external container registry)
5. An Ubuntu based **Google Compute Engine Instance** in the VPC that will be used as a jump host

## Download Software
Download the following artifacts from [Tanzu Network](network.tanzu.vmware.com/) to your jump host.

| Artifact | Version  | Notes |
|---|---|---|
| Cluster Essentials | 1.4.1 | This package containes [Carvel](https://carvel.dev/) toolsand includes binaries to install on your jump host as well as kapp controller which will be deployed to the Kubernetes cluster. |
| Tanzu Framework | 1.4.4 | This is the name of the Tanzu CLI which is the primary interface for platform engineers and application teams to interact with TAP. |

<!--
END: ## Prepare The Infrastructure
-->

## Prepare The Jump Host

Before you proceed, install the following components:

| Software | Version | Check |
|---|---|---|
| Docker | >= 20.10 | `docker version` |
| Kubectl | >= 1.25  | `kubectl version --client` |

### Install Tanzu CLI

1. Extract the downloaded archive
```
mkdir tanzu-framework
tar xvf tanzu-framework.tar -C tanzu-framework
cd tanzu-framework
```

2. Install the executable
```
DIR=$(file cli/core/v* | cut -d ':' -f 1)
sudo install $DIR/tanzu-core-linux_amd64 /usr/local/bin/tanzu
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

### Install Carvel Tools
The binaries we need to have installed are shipping with the previously downloaded Cluster Essentials package.

1. Extract the downloaded archive
```
mkdir cluster-essentials
tar xvf cluster-essentials.tgz -C cluster-essentials
cd cluster-essentials
```

2. Install the executables
```
sudo install ./$I /usr/local/bin/imgpkg
sudo install ./$I /usr/local/bin/kapp
sudo install ./$I /usr/local/bin/kbld
sudo install ./$I /usr/local/bin/ytt
```

3. Validate the installation
```
imgpkg --version
kapp --version
kbld --version
ytt --version
```

<!--
END: ## Prepare The Jump Host
-->

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
  -b registry.tanzu.vmware.com/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:$SHA \
  --to-repo ${HOST}/${REPO}/cluster-essentials-bundle \
  --include-non-distributable-layers
```

3. Mirror TAP packages
```
VERSION="1.4.2"
HOST="gcr.io"
REPO="..."

imgpkg copy \
  -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:$VERSION \
  --to-repo ${HOST}/${REPO}/tap-packages \
  --include-non-distributable-layers
```

<!--
END: ## Create A Package Repository Mirror
-->

## Install Cluster Essentials

1. Setup environment variables
```
GOOGLE_APPLICATION_CREDENTIALS="???"
SHA="2354688e46d4bb4060f74fca069513c9b42ffa17a0a6d5b0dbb81ed52242ea44"
HOST="gcr.io"
REPO="..."

export INSTALL_REGISTRY_HOSTNAME="gcr.io"
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

## Install TAP

TAP is installed via the `tap` meta package. The `tap` package takes a single configuration file (`values.yaml`) as a parameter and passes sections of it to its dependent packages. 

1. Create the configuration file

**values.yaml** (source: [documentation](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.4/tap/install.html#install-your-tanzu-application-platform-profile-2))
```
shared:
  ingress_domain: "INGRESS-DOMAIN"
  
  image_registry:
    project_path: "HOST/REPO"
    username: USERNAME"
    password: "PASSWORD"

  kubernetes_distribution: ""
  kubernetes_version: ""

ceip_policy_disclosed: true

profile: full

supply_chain: basic

ootb_supply_chain_basic:
  registry:
    server: "HOST"
    repository: "REPO"
    
contour:
  envoy:
    service:
      type: LoadBalancer

buildservice:
  kp_default_repository: "REPO"
  kp_default_repository_username: "USERNAME"
  kp_default_repository_password: "PASSWORD"

tap_gui:
  service_type: ClusterIP

metadata_store:
  ns_for_export_app_cert: "default"
  app_service_type: ClusterIP 

scanning:
  metadataStore:
    url: ""

grype:
  namespace: "default"
  targetImagePullSecret: "tap-registry"
```

2. Install the `tap` package with that configuration

```
tanzu package install tap \
    -p tap.tanzu.vmware.com \
    -v "VERSION" \
    --values-file values.yaml \
    --wait="false" \
    -n "tap-install"
```

3. Watch the reconciliation process

```
tanzu -n tap-install packages installed list
kubectl -n tap-install get packageinstalls 
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

## Validate The Installation

### Access TAP GUI
1. Open your browser at [http://tap-gui.DOMAIN](http://tap-gui.DOMAIN)

### Deploy A Test Workload

1. Create a developer namespace
```
kubectl create ns --dry-run=client -o yaml test | kubectl apply -f -
kubectl label namespaces test apps.tanzu.vmware.com/tap-ns=""
```

2. Deploy the workload
```
tanzu app workload create \
  -n test \
  ???
```

<!--
END: ## Validate The Installation
-->