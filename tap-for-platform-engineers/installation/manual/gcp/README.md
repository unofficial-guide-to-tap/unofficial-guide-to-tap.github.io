# Manual Installation On GCP

## GCP Preparations

### GKE Cluster
...

### Container Registry
...

### Service Account
...


## Tanzu Network Downloads
- Download Cluster Essentials (= Carvel)
- Download Tanzu Framework (= Tanzu CLI)

## Prepare Your Machine
Before you can begin installing TAP, you will need to have access so some software on your installation machine. This can be your workstation, yet, the instructions below refer to a Linux machine. Typically, it is advisable to run the installation on a jumphost.

#### Docker
Later in the process, we will mirror container images from Tanzu Network to your own registry. In order to pull images from Tanzu Network, you will need Docker installed locally.

#### Kubectl
Since TAP runs on top of Kubernetes, the main CLI tool to interact with Kubernetes needs to be available. We will need it to prepare, monitor and troubleshoot the installation and it is used behind the scenes by other commands.

#### Tanzu CLI
Tanzu CLI is Tanzu's main command line interface to interact with variety of products. For developers, it can be the one and only abstraction they need while for platform engineers it's a helper for installing and operating TAP.

1. Extract
```
mkdir tanzu-framework
tar xvf tanzu-framework.tar -C tanzu-framework
cd tanzu-framework
```

2. Install executable
```
DIR=$(file cli/core/v* | cut -d ':' -f 1)
sudo install $DIR/tanzu-core-linux_amd64 /usr/local/bin/tanzu
```

3. Install plugins
```
export TANZU_CLI_NO_INIT=true
tanzu plugin install --local cli all
```

4. Validate
```
tanzu version
tanzu plugin list
```

#### Carvel Tools
Carvel exists in multiple separate tools. We will need the the excutables `imgpkg`, `kapp`, `kbld` and `ytt` installed on our installation machine. KApp Controller will later be installed into the cluster.

1. Extract
```
mkdir cluster-essentials
tar xvf cluster-essentials.tgz -C cluster-essentials
cd cluster-essentials
```

2. Install
```
sudo install ./$I /usr/local/bin/imgpkg
sudo install ./$I /usr/local/bin/kapp
sudo install ./$I /usr/local/bin/kbld
sudo install ./$I /usr/local/bin/ytt
```

3. Validate
```
imgpkg --version
kapp --version
kbld --version
ytt --version
```

## Package Repositories

### Docker Login
In order to mirror container images from Tanzu Network to your Google Container Registry, the local Docker service must be logged in to both registries.

#### Tanzu Network
```
TANZUNET_USERNAME="..."
TANZUNET_PASSWORD="..."

echo "$TANZUNET_PASSWORD" | \
  docker login registry.tanzu.vmware.com \
    -u $TANZUNET_USERNAME \
    --password-stdin
```

#### Container Registry
```
gcloud auth login
gcloud auth configure-docker
```

### Cluster Essentials Packages
```
SHA="..."
HOST="..,"
REPO="..."

imgpkg copy \
  -b registry.tanzu.vmware.com/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:$SHA \
  --to-repo ${HOST}/${REPO}/cluster-essentials-bundle \
  --include-non-distributable-layers
```

### TAP Packages
```
VERSION="..."
HOST="..."
REPO="..."

imgpkg copy \
  -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:$VERSION \
  --to-repo ${HOST}/${REPO}/tap-packages \
  --include-non-distributable-layers
```

## Cluster Essentials
This step installs Kapp Controller, which is a component of Carvel, on the Kubernetes cluster. Finally, the previously created package repository mirrors are registered.

1. Setup environment variables
```
HOST="..."
REPO="..."
SHA="..."

export INSTALL_REGISTRY_HOSTNAME="$HOST"
export INSTALL_BUNDLE="$HOST/$REPO/cluster-essentials-bundle@sha256:$SHA"
export INSTALL_REGISTRY_USERNAME="_json_key"
export INSTALL_REGISTRY_PASSWORD="$(cat $GOOGLE_APPLICATION_CREDENTIALS)"
```

Note: The variables above a supposed to point to the mirror of the cluster-essentials-bundle repository that was created in a previous step.

2. Run the installation script
```
cd cluster-essentials
./install.sh --yes
```

## Install TAP

TAP is installed via the `tap` meta package. The `tap` package takes a single configuration file (`values.yaml`) as a parameter and passes sections of it to its dependent packages. 

### Configuration File

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

### Run The Installation

### Validate

```
tanzu package install tap \
    -p tap.tanzu.vmware.com \
    -v "VERSION" \
    --values-file values.yaml \
    --wait="false" \
    -n "tap-install"
```


## Create DNS Records

## Validate

### Access TAP GUI

### Deploy A Test Workload