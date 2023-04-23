# TAP On GCP

## Preparing Google Cloud Platform

Create the following GCP resources manually or use this [Terraform](https://github.com/unofficial-guide-to-tap/terraform/tree/main/gcp) project to automate the setup.

1. Create a **Virtual Private Cloud** (VPC)
2. Create a **Google Kubernetes Engine** (GKE) cluster in the VPC (check official docs for cluster requirements)
3. Create a **Cloud DNS** Managed Zone (or external DNS service)
4. Have access to **Google Container Registry** (GCR) (or external container registry)
5. An Ubuntu based **Google Compute Engine Instance** in the VPC that will be used as a jump host

## Tanzu Network Downloads
Download the following artifacts from [Tanzu Network](network.tanzu.vmware.com/) to your jump host.

**Cluster Essentials 1.4.1**

This package containes [Carvel](https://carvel.dev/) toolsand includes binaries to install on your jump host as well as kapp controller which will be deployed to the Kubernetes cluster.

**Tanzu Framework 1.4.4**

This is the name of the Tanzu CLI which is the primary interface for platform engineers and application teams to interact with TAP.

<!--
END: ## Preparing Google Cloud Platform
-->

## Set Up Your Jump Host

Before you proceed, install the following components:

- Docker (`docker version`)
- Kubectl (`kubectl version --client`)

### Tanzu CLI

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

### Cluster Essentials: Carvel Tools

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
END: ## Set Up Your Jump Host
-->

## Package Repository Mirror

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
SHA="..."
HOST="..,"
REPO="..."

imgpkg copy \
  -b registry.tanzu.vmware.com/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:$SHA \
  --to-repo ${HOST}/${REPO}/cluster-essentials-bundle \
  --include-non-distributable-layers
```

3. Mirror TAP packages
```
VERSION="..."
HOST="..."
REPO="..."

imgpkg copy \
  -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:$VERSION \
  --to-repo ${HOST}/${REPO}/tap-packages \
  --include-non-distributable-layers
```

<!--
END: ## Package Repository Mirror
-->

## Cluster Essentials: Kapp Controller

1. Setup environment variables
```
HOST="HOST_OF_YOUR_PACKAGE_MIRROR_REGISTRY"
REPO="REPO_OF_YOUR_PACKAGE_MIRROR_REGISTRY"
SHA="SHA_HASH_OF_THE_INSTALLATION_BUNDLE" ???

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
END: ## Cluster Essentials: Kapp Controller
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

## Create DNS Records

1. Get the public IP of your Contour ingress (installed with TAP)
```
kubectl -n tanzu-system-ingress \
  get svc envoy \
  -o json \
  | jq ".status.loadBalancer.ingress[] | .ip" -r
```

2. Create A record for `*.DOMAIN` pointing to that address

The [Terraform](https://github.com/unofficial-guide-to-tap/terraform/tree/main/gcp) project mentioned above allows you to configure the IP address as a variable so you can manage these entries via Terraform.

<!--
END: ## Install TAP
-->

## Validate

### Access TAP GUI

### Deploy A Test Workload