# Manual Installation On GCP

## Create the GKE Cluster
...

## Tanzu Network Downloads
- Download Cluster Essentials (= Carvel)
- Download Tanzu Framework (= Tanzu CLI)


## Prepare Your Machine

#### Docker
...

#### Kubectl
- Make sure to have kubectl installed
- Version that matches the K8s cluster

#### Tanzu CLI
1. Extract the downloaded TAR file:
```
mkdir tanzu-framework
tar xvf tanzu-framework.tar -C tanzu-framework
cd tanzu-framework
```

2. Find the executable and install it locally:
```
export TANZU_CLI_NO_INIT=true
DIR=$(file cli/core/v* | cut -d ':' -f 1)

sudo install $DIR/tanzu-core-linux_amd64 /usr/local/bin/tanzu
tanzu plugin install --local cli all
```

3. Validate the installation:
```
tanzu version
tanzu plugin list
```

#### Carvel Tools

1. Extract the TGZ file
```
mkdir cluster-essentials
tar xvf cluster-essentials.tgz -C cluster-essentials
```

2. Install carvel executables locally:
```
cd cluster-essentials
for I in imgpkg kapp kbld ytt; do 
  sudo install ./$I /usr/local/bin/$I
  $I --version
done
```

## Package Repositories

### Cluster Essentials Packages
```
```

### TAP Packages
```
```


## Cluster Essentials
This step installs KApp Controller, which is a component of Carvel, on the Kubernetes cluster. Finally, the previously created package repository mirrors are registered.




## Install TAP

### Configuration File

### Install The Packages


### Create DNS Records