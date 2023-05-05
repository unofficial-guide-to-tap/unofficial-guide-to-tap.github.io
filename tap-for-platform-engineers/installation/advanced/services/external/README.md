# External Services

Not all organizations are flexible enough to switch the management of data services (such as databases) onto Kubernetes and into a developer platform the day TAP is introduced. For situations in which an organization's processes take more time, you will be wondering _How can I still create a good developer experience?_ 

With Services Toolkit, TAP already provides the mechanisms necessary to allows application teams to *discover* services available to them, *claim* instances of that service and *bind* that services instance to their application. What we need is a way to hook an externally managed service into that mechanism.

In this guide, we will run through this at the example of a **PostgreSQL** database.

## Create A Secret

To make use of Services Toolkit, we need a Kubernetes resource in the cluster that represents the external service. This is done by creating a `Secret` that is compatible to the service binding specification. 

1. Create a `Namespace` to host all service resources

    ```
    kubectl create namespace service-instances
    ```

2. Create a `Secret` to represent the external service

    ```
    cat <<EOF | kubectl apply -f -
    apiVersion: v1
    kind: Secret
    metadata:
    name: postgres-1
    type: servicebinding.io/mysql
    stringData:
      type: postgres
      provider: myorg
      host: 192.168.234.21
      port: 5432
      username: appuser
      password: apppass
    EOF
    ```

## Expose The Service

Next, we use Services Toolkit to expose the service instance to application teams allowing them to discover and claim the service instance. There are 2 approaches:

### Direct Claims

In this method, we create a `ResourceClaimPolicy` telling Services Toolkit about the `Secret` that represents the service. Using the policy, we can choose the developer `Namespace`s that may discover and claim the service instance or expose it to all `Namespace`s using a wildcard (`*`).

```
cat <<EOF | kubectl apply -f -
apiVersion: services.apps.tanzu.vmware.com/v1alpha1
kind: ResourceClaimPolicy
metadata:
  name: postgres-1-to-all
  namespace: service-instances
spec:
  consumingNamespaces:
  - '*'
  subject:
    group: v1
    kind: Secret
    name: postgres-1
EOF
```

**Validation**

In order to create a `ResourceClaim`, the application team needs to know the exact specification and `kubectl apply` the YAML themselves:

```bash
tanzu service claim create postgres-db \
  --resource-name postgres-1 \
  --resource-namespace service-instances \
  --resource-kind Secret \
  --resource-api-version v1
```

### Service Class

This approach is less specific when it comes to mapping services instances to applicatin teams. It presents a service class (`ClusterInstanceClass`) and adds our service instances to a pool of services. Application teams then claim the class rather than an individual instance and receive a free service from that pool.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: services.apps.tanzu.vmware.com/v1alpha1
kind: ClusterInstanceClass
metadata:
name: ext-postgres
spec:
description:
  short: PostgreSQL managed by our DBAs
pool:
  group: v1
  kind: Secret
  ???
EOF
```

TODO: Create role

**Validation**

App teams may now discover available classes and claim instances from the pool:

```bash
tanzu service classes list
```
Expected output:
```
TODO
```

```bash
tanzu service claimable list --class ext-postgres
```

Expected output:
```
TODO
```