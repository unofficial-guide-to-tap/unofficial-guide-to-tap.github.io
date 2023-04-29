# TAP on Minikube (NOT WORKING)

## Prerequisites

The following requirements have been tested and are known to work. You might be using a different operating system or different versions of software or a different driver for Minikube. In this case, you will need to make adjustments to the instructions below.

| Prequisite | Version |
|---|---|
| Operating System | Mac OS Ventura 13.3.1, M1
| Container Runtime | Docker Desktop 4.16.1 |
| Minikube | 1.30.1 |

You will also need

2. **Tanzu Network:** Access to [Tanzu Network](https://network.tanzu.vmware.com) to download TAP and other components

3. **Docker Hub:** TAP will use Docker Hub as a registry to store application images.

## Create The Minikube Cluster
```
minikube start --kubernetes-version='1.25.9' --cpus='8' --memory='32g' --driver='docker'
```

## Download Software

Download the following artifacts from [Tanzu Network](https://network.tanzu.vmware.com/) to your jump host.

| Artifact | Version  | Notes |
|---|---|---|
| Cluster Essentials | 1.4.1 | Contains [Carvel](https://carvel.dev/) tools |
| Cluster Essentials Bundle YAML | 1.4.1 | Contains the SHA hash |
| Tanzu Framework | 1.4.4 | This is the name of the Tanzu CLI which is the primary interface for platform engineers and application teams to interact with TAP. |

## Prepare Your Workstation

### Install Tanzu CLI

1. Extract the downloaded archive
    ```
    mkdir tanzu-framework
    tar xvf tanzu-framework-darwin-amd64-v0.25.4.6.tar -C tanzu-framework
    cd tanzu-framework
    ```

2. Install the executable
    ```
    sudo install cli/core/v0.25.4/tanzu-core-darwin_amd64 /usr/local/bin/tanzu
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
    tar xvf tanzu-cluster-essentials-darwin-amd64-1.4.1.tgz -C cluster-essentials
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

4. Install `kctrl` (not shipped with Cluster Essentials)

    1. Navigate to [https://github.com/carvel-dev/kapp-controller/releases](https://github.com/carvel-dev/kapp-controller/releases)
    2. Download the binary that applied to your operating system
    3. Install the binary

    ```
    sudo install kctrl-darwin-arm64 /usr/local/bin/kctrl
    ```

    4. Validate `kctrl`
    ```
    kctrl version
    ```

## Install Cluster Essentials

1. Setup environment variables
    
    ```
    TANZUNET_HOST="registry.tanzu.vmware.com"
    TANZUNET_USER="..."
    TANZUNET_PASS="..."
    SHA="2354688e46d4bb4060f74fca069513c9b42ffa17a0a6d5b0dbb81ed52242ea44"
    ```
    ```
    export INSTALL_REGISTRY_HOSTNAME="$TANZUNET_HOST"
    export INSTALL_BUNDLE="$TANZUNET_HOST/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:$SHA"
    export INSTALL_REGISTRY_USERNAME="$TANZUNET_USER"
    export INSTALL_REGISTRY_PASSWORD="$TANZUNET_PASS"
    ```

2. Run the installation script
    ```
    cd cluster-essentials
    ./install.sh --yes
    ```

## Install The TAP Package

### Prepare The Installation Namespace

1. Create the installation `Namespace`
    ```
    kubectl create namespace tap-install
    ```

2. Create registry `Secret`

    ```
    TANZUNET_HOST="registry.tanzu.vmware.com"
    TANZUNET_USER="..."
    TANZUNET_PASS="..."
    ```

    ```
    tanzu secret registry add tap-registry \
      --username "$TANZUNET_USER" \
      --password "$TANZUNET_PASS" \
      --server "$TANZUNET_HOST" \
      --export-to-all-namespaces --yes --namespace tap-install
    ```

2. Create `PackageRepository`

    ```
    TAP_VERSION="1.4.4"
    TANZUNET_HOST="registry.tanzu.vmware.com"

    tanzu package repository add tanzu-tap-repository \
      --namespace tap-install \
      --url ${TANZUNET_HOST}/tanzu-application-platform/tap-packages:${TAP_VERSION}
    ```

3. Create the TAP configuration file

    ```
    vim values.yaml
    ```
    ```
    shared:
      ingress_domain: YOUR_DOMAIN

      image_registry:
        project_path: "index.docker.io/DOCKER_REPO"
        username: "DOCKER_USERNAME"
        password: "DOCKER_PASSWORD"

    ceip_policy_disclosed: true

    profile: full

    supply_chain: basic

    contour:
      envoy:
        service:
          type: LoadBalancer

    tap_gui:
      service_type: ClusterIP

    cnrs:
      domain_template: "{{.Name}}-{{.Namespace}}.{{.Domain}}"
    ```

4. Install the `tap` package with that configuration

    ```
    tanzu package install tap \
      -p tap.tanzu.vmware.com \
      -v "$TAP_VERSION" \
      --values-file values.yaml \
      --wait="false" \
      -n "tap-install"
    ```

3. Watch progress of the `PackageInstall`s

    ```
    tanzu -n tap-install package installed list
    ```

## Troubleshooting

### Reconciliation Failed
Container images will be pulled directly from Tanzu Network. It might take a while until all packages are reconciled depending on the power of your Minikube instance and your local internet connection. Kapp Controller is likely to time out on the package reconciliation process and show `Reconcile failed`. That's okay, it took over an hour in our tests until the packages have recovered. You may use `kubectl get pods -A` to check for `Pod`s with a status of "ContainerCreating" to identify `Package`s that need more time.

