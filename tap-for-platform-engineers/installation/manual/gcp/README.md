# TAP On GCP

<!-- TOC depthfrom:2 depthto:2 orderedlist:false -->

- [Prepare The Infrastructure](#prepare-the-infrastructure)
- [Download Software](#download-software)
- [Prepare The Jump Host](#prepare-the-jump-host)
- [Create A Package Repository Mirror](#create-a-package-repository-mirror)
- [Install Cluster Essentials](#install-cluster-essentials)
- [Install TAP](#install-tap)
- [Create DNS Records](#create-dns-records)
- [Validate The Installation](#validate-the-installation)

<!-- /TOC -->

## Prepare The Infrastructure

Create the following GCP resources manually or use this [Terraform](https://github.com/unofficial-guide-to-tap/terraform/tree/main/gcp) project to automate the setup.

1. Create a **Virtual Private Cloud** (VPC)
2. Create a **Google Kubernetes Engine** (GKE) cluster in the VPC (check official docs for cluster requirements)
3. Create a **Cloud DNS** Managed Zone (or external DNS service)
4. Have access to **Google Container Registry** (GCR) (or external container registry)
5. A **Service Account Key** to be used by TAP 
6. An Ubuntu based **Google Compute Engine Instance** in the VPC that will be used as a jump host

## Download Software
Download the following artifacts from [Tanzu Network](network.tanzu.vmware.com/) to your jump host.

| Artifact | Version  | Notes |
|---|---|---|
| Cluster Essentials for Linux| 1.4.1 | Contains [Carvel](https://carvel.dev/) tools |
| Cluster Essentials Bundle YAML | 1.4.1 | Contains the SHA hash |
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
| gcloud | latest  | `gcloud version` |

### Connect To Kubernetes

```
gcloud auth login
gcloud container clusters get-credentials CLUSTER_NAME \
  --region REGION \
  --project PROJECT_ID
```

### Service Account Key

Create a copy of the Service Account key at `$HOME/key.json`.
```
vim $HOME/key.json
```

### Install Tanzu CLI

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

### Install Carvel Tools
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

## Install TAP

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
  ingress_domain: "INGRESS-DOMAIN"
  
  image_registry:
    project_path: "HOST/REPO"
    username: "_json_key"
    password: |
      SERVICE_ACCOUNT_KEY_JSON

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

cnrs:
  domain_template: "{{.Name}}-{{.Namespace}}.{{.Domain}}"

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

## Validate The Installation

### Access TAP GUI

1. Open your browser (ideally, use Google Chrome as it allows you to accept the self-signed certificate)
2. Navigate to [https://tap-gui.DOMAIN](http://tap-gui.DOMAIN)
3. In the "Guest" panel, click "ENTER" to proceed as a guest user


### Deploy A Test Workload

1. Create a developer namespace
```
kubectl create ns --dry-run=client -o yaml test | kubectl apply -f -
kubectl label namespaces test apps.tanzu.vmware.com/tap-ns=""
```

2. Deploy the workload
```
tanzu apps workload create petclinic -n test \
  -l "app.kubernetes.io/part-of=petclinic" \
  -l "apps.tanzu.vmware.com/workload-type=web" \
  --build-env "BP_JVM_VERSION=17" \
  --git-repo https://github.com/spring-projects/spring-petclinic.git \
  --git-branch main
```

3. Monitor progress

Run the following command and check the readiness and health state of supply chain resources:
```
tanzu apps workload get petclinic -n test
```

Tail the logs of the supply chain execution:
```
tanzu apps workload tail petclinic --namespace test
```

Watch the progress in the UI:
1. Open your browser at [https://tap-gui.DOMAIN](http://tap-gui.DOMAIN)
2. In the menu, navigate to "Supply Chains"
3. Click on `petclinic`

<!--
END: ## Validate The Installation
-->