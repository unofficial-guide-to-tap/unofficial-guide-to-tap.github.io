# GitOps Installation On Google Cloud Platform

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
gcloud container clusters get-credentials CLUSTER_NAME \
  --region REGION \
  --project PROJECT_ID
```

#### Service Account Key

Create a copy of the Service Account key at `$HOME/key.json`.
```bash
vim $HOME/key.json
```

### Download Software
Download the following artifacts from [Tanzu Network](https://network.tanzu.vmware.com/) to your jump host.

| Artifact | Version  | Notes |
|---|---|---|
| Cluster Essentials for Linux | 1.5. | Contains [Carvel](https://carvel.dev/) tools |
| Cluster Essentials Bundle YAML | 1.5 | Contains the SHA hash |
| Tanzu Framework | 1.5.0 | This is the name of the Tanzu CLI which is the primary interface for platform engineers and application teams to interact with TAP. The software bundle is part of the "Tanzu Application Platform" product in Tanzu Network |
| Tanzu GitOps Reference Implementation | 1.5.0 | An archive with the contents to intitialize a gitops repository for TAP |


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
    TANZUNET_USERNAME="..."
    TANZUNET_PASSWORD="..."

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
    # gcloud auth login # If you haven't already done so
    gcloud auth configure-docker
    ```

3. Mirror cluster essentials packages

    The SHA hash used in this section is taken from the file `foo` that you downloaded before. In case you're using a different version, you may extract that hash from the yaml with the following command:

    ```bash
    cat tanzu-cluster-essentials-bundle-1.5.0.yml | yq '.bundle.image' | cut -d ":" -f 2
    ```

    ```bash
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
    ```bash
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

### Install Cluster Essentials

1. Setup environment variables
    ```bash
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
    ```bash
    cd cluster-essentials
    ./install.sh --yes
    ```

<!--
END: ## Install Cluster Essentials
-->

### Install SOPS CLI

1. Download and install the `sops` binary

    ```bash
    wget https://github.com/mozilla/sops/releases/download/v3.7.3/sops-v3.7.3.linux.amd64
    sudo install sops-v3.7.3.linux.amd64 /usr/local/bin/sops
    ```

2. Verify `sops` installation and version
    ```
    sops --version
    ```
    Expected output
    ```
    sops 3.7.3 (latest)
    ```

<!--
END: ### Install SOPS CLI
-->

### Install Age CLI

1. Download and install the `age` binary

    ```bash
    sudo apt install age
    ```

2. Verify `sops` installation and version
    ```
    age --version
    ```
    Expected output
    ```
    1.0.0
    ```

<!--
END: ### Install Age CLI
-->

### Setup The Git Repository

#### Initialize The Repo

1. Create Git Repository

    Go to Github, Gitlab, Bitbucket or whatever service you use to host Git repositories and create a new empty repository. The following instructions are designed for **GitHub** repositories

2. Initialize the repository

    Create a folder for the repository
    ```
    mkdir tap-gitops
    ```

    Unpack the "Tanzu GitOps Reference Implementation" archive into the folder

    ```
    tar xvf tanzu-gitops-ri-0.1.0.tgz -C tap-gitops
    ```

    Create cluster config

    ```
    pushd tap-gitops
    ./setup-repo.sh tap-cluster sops
    ````

    Push changes

    ```
    git init
    git add .
    git commit -m "Add full-tap-cluster"
    git branch -M main
    git remote add origin git@github.com:mathias-ewald/tap-gitops.git
    git push -u origin main
    ```

    Running `tree` in the `tap-gitops` folder should reveal a structure like this:

    ```
    .
    ├── README.md
    ├── clusters
    │   └── tap-cluster
    │       ├── cluster-config
    │       │   ├── config
    │       │   │   └── tap-install
    │       │   └── values
    │       ├── docs
    │       │   └── README.md
    │       └── tanzu-sync
    │           ├── app
    │           │   ├── config
    │           │   └── values
    │           ├── bootstrap
    │           │   ├── noop
    │           │   └── readme
    │           └── scripts
    │               ├── configure-secrets.sh
    │               ├── configure.sh
    │               ├── deploy.sh
    │               └── sensitive-values.sh
    └── setup-repo.sh
    ```

#### Create Encrypted File For Sensitive Values

1. Create a temporary directory to setup the encryption keys

    ```
    mkdir -p tmp-enc
    chmod 700 tmp-enc
    cd tmp-enc
    ```

2. Create the key

    ```
    age-keygen -o key.txt
    ```

3. Create and encrypt the yaml file for senstive values

    ```
    cat <<EOF > tap-sensitive-values.yaml
    ---
    tap_install:
      sensitive_values:
        shared:
          image_registry:
            password: |
              SERVICE_ACCOUNT_KEY_JSON
    EOF

    export SOPS_AGE_RECIPIENTS=$(cat key.txt | grep "# public key: " | sed 's/# public key: //')
    sops --encrypt tap-sensitive-values.yaml > tap-sensitive-values.sops.yaml
    ```

4. Move the encrypted file into the `tap-gitops` repository

    ```
    mv tap-sensitive-values.sops.yaml ~/tap-gitrepo/clusters/<CLUSTER-NAME>/cluster-config/values/
    ```

5. Save `key.txt` and cleanup

    ```
    cp key.txt $HOME/key.txt
    cd $HOME
    rm -fR tmp-enc
    ```

If you need to change your sensitive values later on, simple make sure to export `SOPS_AGE_RECIPIENTS` pointing your your `key.txt` and run `sops tap-sensitive-values.sops.yaml`. This will open an editor with the unencrypted contents. Make your changes, then save the file.

#### Create File For Non-Sensitive Values

```
cd tap-gitops/clusters/tap-cluster/cluster-config/values
```
```
cat <<EOF > tap-non-sensitive-values.yaml
---
tap_install:
  values:
    shared:
      ingress_domain: YOUR_DOMAIN
      image_registry:
        project_path: "REGISTRY_HOST/REGISTRY_REPO"
        username: "_json_key"

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

### Tanzu Sync

#### Generate The Tanzu Sync Configuration

1. Set up environment variables

```
export INSTALL_REGISTRY_HOSTNAME="gcr.io"
export INSTALL_REGISTRY_USERNAME="_json_key"
export INSTALL_REGISTRY_PASSWORD="$(cat $HOME/key.json | jq -c | jq -R | sed 's/^"//' | sed 's/"$//')"
export GIT_SSH_PRIVATE_KEY="$(cat $HOME/.ssh/id_rsa)"
export GIT_KNOWN_HOSTS="$(ssh-keyscan github.com)"
export SOPS_AGE_KEY="$(cat $HOME/key.txt)"
export TAP_PKGR_REPO="gcr.io/cso-pcfs-emea-mewald/tap-packages"
```

2. Run `tanz-sync`'s `configure.sh` script

```
cd tap-gitops/clusters/tap-cluster
./tanzu-sync/scripts/configure.sh
```

3. Push changes

```
git add cluster-config/ tanzu-sync/
git commit -m "Configure install of TAP 1.5.0"
git push
```

#### Deploy Tanzu Sync

```
cd tap-gitops/clusters/tap-cluster
./tanzu-sync/scripts/deploy.sh
```


<!--
END: ### Setup The Git Repository
-->


### Create DNS Records

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

<!--
END: # Installation
-->

## Validation <!-- omit from toc -->

### Access TAP GUI

1. Open your browser (ideally, use Google Chrome as it allows you to accept the self-signed certificate)
2. Navigate to [https://tap-gui.DOMAIN](http://tap-gui.DOMAIN)
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
