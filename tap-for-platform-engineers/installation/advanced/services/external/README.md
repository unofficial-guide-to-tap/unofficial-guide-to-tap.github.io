# External Services

Not all organizations are flexible enough to switch the management of data services (such as databases) onto Kubernetes and into a developer platform the day TAP is introduced. For situations in which an organization's processes take more time, you will be wondering _How can I still create a good developer experience?_ 

With Services Toolkit, TAP already provides the mechanisms necessary to allows application teams to *discover* services available to them, *claim* instances of that service and *bind* that services instance to their application. What we need is a way to hook an externally managed service into that mechanism.

In this guide, we will run through this at the example of an external **PostgreSQL** database.

## Register The Service

To make use of Services Toolkit, we need a Kubernetes resource in the cluster that represents the external service. This is done by creating a `Secret` that is compatible to the service binding specification. 

1. Create a `Namespace` to host all service resources

    ```
    kubectl create namespace service-instances
    ```

2. Create a `Secret` to represent the external service

    ```
    cat <<EOF | kubectl -n service-instances apply -f -
    apiVersion: v1
    kind: Secret
    type: servicebinding.io/mysql
    metadata:
      name: postgres-1
    stringData:
      type: "postgres"
      provider: "myorg"
      host: "192.168.234.21"
      port: "5432"
      username: "appuser"
      password: "apppass"
    EOF
    ```

## Expose The Service

Next, we use Services Toolkit to expose the service instance to application teams allowing them to discover and claim the service instance. Before we dive deeper into any of the two approaches to achieve this, we need to make sure Services Toolkit has access to `Secrets` in the `service-instances` `Namespace`: 

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: access-service-instance-secrets
  labels:
    servicebinding.io/controller: "true"
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch"]
EOF
```

### Direct Claims

In this method, we create a `ResourceClaimPolicy` telling Services Toolkit about the `Secret` that represents the service. Using the policy, we can choose the developer `Namespace`s that may discover and claim the service instance or expose it to all `Namespace`s using a wildcard (`*`).

#### Configuration

Create a `ResourceClaimPolicy` that allow to consume the service from all `Namespaces`:

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
    resourceNames:
    - postgres-1
EOF
```

#### Validation

In order to create a `ResourceClaim`, the application team needs to know the exact specification and `kubectl apply` the YAML themselves:

```bash
tanzu service claim create postgres-db \
  --resource-name postgres-1 \
  --resource-namespace service-instances \
  --resource-kind Secret \
  --resource-api-version v1
```

### Service Class

#### Configuration

This approach is less specific when it comes to mapping services instances to applicatin teams. It presents a service class (`ClusterInstanceClass`) and adds our service instances to a pool of services. Application teams then claim the class rather than an individual instance and receive a free service from that pool.

1. Label the `Secret`

  Since we're using plain `Secrets` to represent our external services, Kubernetes cannot distinguish it from any other non-postgres services. In order to create a class consisting of only externally managed Postgres instances, we need to add a label to the `Secret`:

  ```bash
  kubectl patch secret -n service-instances postgres-1 --patch '{"metadata": {"labels": {"postgres": "ext"}}}'
  ```


2. Create a `ClusterInstanceClass` for Postgres services

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
      kind: Secret
      labelSelector:
        matchLabels:
          postgres: ext
  EOF
  ```

#### Validation

App teams may now discover available classes and claim instances from the pool:

```bash
tanzu service classes list
```
Expected output:
```
NAME                  DESCRIPTION
ext-postgres          PostgreSQL managed by our DBAs
```

```bash
tanzu service claimable list --class ext-postgres --namespace service-instances
```

Expected output:
```
NAME        NAMESPACE          KIND    APIVERSION
postgres-1  service-instances  Secret  v1
```