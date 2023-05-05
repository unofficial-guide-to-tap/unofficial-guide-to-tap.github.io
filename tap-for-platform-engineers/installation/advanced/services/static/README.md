# Static Provisioning

Technically, working with [External Services](tap-for-platform-engineers/installation/advanced/services/external/README.md) is a form of static provisioning with the caveat that service instances aren't actually provisioned on Kubernetes. In that module, we viewed service instances as external factor and demonstrated how to hook them into the machanics of Services Toolkit by creating a `Secret`. In this module, we want to take a look at actually provisioning service instances on Kubernetes and what that looks like integrating into Services Toolkit. 

## Deploying Services

Before we can think about exposing those services and making them claimable for app teams, we need to extend our Kubernetes cluster with the capability to provision services through Kubernetes's declarative API. The same way we create a `Pod`, a `PersistentVolume` or an `HttpProxy`, we want to be able to create a `Postgres`, `MySQL` or `RabbitmqCluster` and have Kubernetes - or rather the respective controller - to go out and take care of the actual provisioning. The creation of that resource can happen dynamically (see module on [Dynamic Provisioning](tap-for-platform-engineers/installation/advanced/services/dynamic/README.md) or statically (by hand) as we're going to cover in this module.

There are many different extensions out there, some of which are a bit simple to integrate than others. If an extension produces resources compatible to the service binding specification, the integration is a breeze. Others require some extra work.

To proceed, you should have one of the following Kubernetes extensions set and configured:

| Extension | Included in TAP | Service Binding Compatible | Guides |
|---|---|---|---|
| Crossplane | yes | ? | * [Install The Crossplane GCP Provider](tap-for-platform-engineers/installation/advanced/services/crossplane-gcp.md)</br>* [Install The Crossplane AWS Provider](tap-for-platform-engineers/installation/advanced/services/crossplane-aws.md)</br>* [Install The Crossplane Azure Provider](tap-for-platform-engineers/installation/advanced/services/crossplane-azr.md) |
| Tanzu SQL | no | yes | * [Install Tanzu SQL](tap-for-platform-engineers/installation/advanced/services/tanzu-sql.md) |
| Tanzu RabbitMQ | no | yes | * [Install Tanzu RabbitMQ](tap-for-platform-engineers/installation/advanced/services/tanzu-rmq.md) |

The rest of this guide, assumes you followed one of the installation guides for extenstions and are able to provision service instances into the `service-instances` `Namespace`.

## Expose The Service

Like with [External Services](tap-for-platform-engineers/installation/advanced/services/external/README.md) there are 2 options to expose statically provisioned services: 

1. Via `ResourceClaimPolicy` and a direct claim
2. As part of the pool of a service class (`ClusterInstanceClass`)

This guide will not document method 1 again as it is the least preferrable one. Instead, we will focus on service class at the example of a PostgreSQL service instance represented by a `Postgres` resource that looks as follows:

```bash
kubectl get postres -n service-instances pg-1 -o yaml
```

```yaml
apiVersion: sql.tanzu.vmware.com/v1
kind: Postgres
metadata:
  name: pg-1
  namespace: service-instances
  ...
spec:
  postgresVersion:
    name: postgres-15
  storageSize: 1G
  ...
status:
  binding:
    name: pg-1-app-user-db-secret
  ...
```

As you can see by the `status.binding.name` attribute, this `Postgres` resource is service binding compatible. Jackpot!

1. Allow Services Toolkit to see `Postgres` resources

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: access-service-instance-postgres
      labels:
        servicebinding.io/controller: "true"
    rules:
      - apiGroups: ["sql.tanzu.vmware.com"]
        resources: ["postgres"]
        verbs: ["get", "list", "watch"]
    EOF
    ```

2. Create a `ClusterInstanceClass`

    The following example create a `ClusterInstanceClass` that pools all available `Postgres` resources. Alternatively, you may narrow down this filter using label and field selectors. Check the documentation for details.

  ```bash
  cat <<EOF | kubectl apply -f -
  apiVersion: services.apps.tanzu.vmware.com/v1alpha1
  kind: ClusterInstanceClass
  metadata:
    name: tanzu-postgres
  spec:
    description:
      short: Tanzu PostgreSQL
    pool:
      group: sql.tanzu.vmware.com
      kind: Postgres
  EOF
  ```


## Validate

App teams may now discover available classes:

```bash
tanzu service classes list
```
Expected output:
```
NAME                  DESCRIPTION
tanzu-postgres        Tanzu PostgreSQL
```

Knowing the class name, a user may now list available service instances in the class:

```bash
tanzu service claimable list --class tanzu-postgres --namespace service-instances
```

Expected output:
```
NAME        NAMESPACE          KIND    APIVERSION
postgres-1  service-instances  Secret  v1
```