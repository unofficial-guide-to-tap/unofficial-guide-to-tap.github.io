# Tanzu RabbitMQ for Kubernetes

Make sure set up the folloring variables before proceeding with this guide:

```
RABBITMQ_VERSION="1.4.0"
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

## Installation

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
      --namespace tanzu-rabbitmq-operator
    ````

4. Install the `Package` for the RabbitMQ operator

    ```
    cat <<EOF > rabbitmq-operator.yaml
    ---
    EOF
    ```

    ```
    tanzu package install rabbitmq-operator \
      --package-name rabbitmq.tanzu.vmware.com \
      --version $RABBITMQ_VERSION \
      --namespace tanzu-rabbitmq-operator \
      -f rabbitmq-operator.yaml
    ```

## Validation

Check availability of CRDs:

```
kubectl api-resources | grep rabbitmq
```

Expected output:
```
bindings                                                                 rabbitmq.com/v1beta1                                                true         Binding
exchanges                                                                rabbitmq.com/v1beta1                                                true         Exchange
federations                                                              rabbitmq.com/v1beta1                                                true         Federation
permissions                                                              rabbitmq.com/v1beta1                                                true         Permission
policies                                                                 rabbitmq.com/v1beta1                                                true         Policy
queues                                                                   rabbitmq.com/v1beta1                                                true         Queue
rabbitmqclusters                  rmq                                    rabbitmq.com/v1beta1                                                true         RabbitmqCluster
schemareplications                                                       rabbitmq.com/v1beta1                                                true         SchemaReplication
shovels                                                                  rabbitmq.com/v1beta1                                                true         Shovel
superstreams                                                             rabbitmq.com/v1alpha1                                               true         SuperStream
users                                                                    rabbitmq.com/v1beta1                                                true         User
vhosts                                                                   rabbitmq.com/v1beta1                                                true         Vhost
standbyreplications               sr                                     rabbitmq.tanzu.vmware.com/v1beta1                                   true         StandbyReplication
```

## Example Usage

1. Create a `Namespace` to run the service instances

    ```
    kubectl create namespace service-instances
    ```

2. Create a `Secret` to access RabbitMQ related images

    ```
    kubectl create secret docker-registry regsecret \
      --namespace service-instances \
      --docker-server=$INSTALL_REGISTRY_HOSTNAME \
      --docker-username="$INSTALL_REGISTRY_USERNAME" \
      --docker-password="$INSTALL_REGISTRY_PASSWORD"
    ```

3. Deploy an instance of `RabbitmqCluster`

    ```
    cat <<EOF | kubectl -n service-instances apply -f -
    apiVersion: rabbitmq.com/v1beta1
    kind: RabbitmqCluster
    metadata:
      name: rmq-1
    EOF
    ```

4. Workaround: The RabbitMQ `Package` currently cannot be parameterized to support private registries. Therefore, we need to add the previously created `Secret` `regsecret` to the `imagePullSecrets` of the `ServiceAccount` running the related `Pods`.

    ```
    kubectl -n service-instances patch serviceaccount rmq-1-server\
      -p '{"imagePullSecrets": [{"name": "regsecret"}]}'
    ```

5. After a while, the resources should have deployed successfully. Run the following command to verify:

    ```
    kubectl -n service-instances get rabbitmqclusters,pods,services
    ```

    Expected output:

    ```
    NAME                                 ALLREPLICASREADY   RECONCILESUCCESS   AGE
    rabbitmqcluster.rabbitmq.com/rmq-1   True               True               81s

    NAME                 READY   STATUS    RESTARTS   AGE
    pod/rmq-1-server-0   1/1     Running   0          53s

    NAME                           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                        AGE
    service/rmq-1                  ClusterIP   10.3.241.57    <none>        15692/TCP,5672/TCP,15672/TCP   81s
    service/rmq-1-nodes            ClusterIP   None           <none>        4369/TCP,25672/TCP             81s
    ```

---
Next: [Services Toolkit](./services-toolkit.md)