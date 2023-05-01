# Tanzu RabbitMQ for Kubernetes

Make sure set up the folloring variables before proceeding with this guide:

```
RABBITMQ_VERSION="1.3.0"
TANZUNET_USERNAME="..."
TANZUNET_PASSWORD="..."

INSTALL_REGISTRY_HOSTNAME="..."
INSTALL_REGISTRY_REPO="..."
INSTALL_REGISTRY_USERNAME="..."
INSTALL_REGISTRY_PASSWORD="..."
```
- **RABBITMQ_VERSION**: The version of Tanzu RabbitMQ as available in Tanzu Network
- **TANZUNET_USERNAME**: The username to authenticate with Tanzu Network. 
- **TANZUNET_PASSWORD**: The password to authenticate with Tanzu Network. 
- **INSTALL_REGISTRY_HOSTNAME**: The hostname of your registry you use as a package mirror of Tanzu Network
- **INSTALL_REGISTRY_REPO**: The repository in that registry.
- **INSTALL_REGISTRY_USERNAME**: The username to authenticate with the registry. 
- **INSTALL_REGISTRY_PASSWORD**: The password to authenticate with the registry. 

## Relocate Images

1. Docker login to Tanzu Network
    ```
    echo "$TANZUNET_PASSWORD" | \
    docker login registry.tanzu.vmware.com \
      -u $TANZUNET_USERNAME \
      --password-stdin
    ```

2. Docker login to your container registry
    ```
    echo "$INSTALL_REGISTRY_PASSWORD" | \
    docker login $INSTALL_REGISTRY_HOSTNAME \
      -u $INSTALL_REGISTRY_USERNAME \
      --password-stdin
    ```

3. Mirror the packages

    ```
    imgpkg copy \
      -b registry.tanzu.vmware.com/p-rabbitmq-for-kubernetes/tanzu-rabbitmq-package-repo:$RABBITMQ_VERSION \
      --to-repo $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REGISTRY_REPO/rmq-packages
    ```

## Install Tanzu RabbitMQ

1. Create the `Namespace` for the operator

    ```
    kubectl create ns tanzu-rabbitmq-operator
    ```

2. Set up a `Secret` with credentials to the container registry containing the `Package`s
    ```
    kubectl create secret docker-registry regsecret \
      --namespace tanzu-rabbitmq-operator \
      --docker-server=$INSTALL_REGISTRY_HOSTNAME \
      --docker-username="$INSTALL_REGISTRY_USERNAME" \
      --docker-password="$INSTALL_REGISTRY_PASSWORD"
    ```
    
3. Register the `PackageRepository`

    ```
    tanzu package repository add tanzu-rabbitmq-repository \
      --url $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REGISTRY_REPO/rmq-packages:$RABBITMQ_VERSION \
      --namespace tanzu-rabbitmq-operator \
    ````

4. Install the `Package` for the RabbitMQ operator

    ```
    cat <<EOF > rabbitmq.yaml
    ---
    EOF
    ```
    ```
    tanzu package install rabbitmq-operator \
      --package-name rabbitmq.tanzu.vmware.com \
      --version $RABBITMQ_VERSION \
      --namespace tanzu-rabbitmq-operator \
      -f rabbitmq.yaml
    ```


---
Next: [Services Toolkit](./services-toolkit.md)