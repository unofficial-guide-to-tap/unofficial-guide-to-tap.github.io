# Services Toolkit

TAP 1.5 includes very detailed [documentation](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/services-toolkit-about.html) on Services Toolkit, its concepts and capabilities. Especially the chapter on the [levels of service consumption](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/services-toolkit-concepts-service-consumption.html) should be read throughly before proceeding.

If you reached this chapter by following the guide step by step, you now have the following:

1. TAP running on a single node Kubernetes cluster
2. A Kubernetes extension that allows you to lifecycle services (Tanzu SQL, Tanzu RabbitMQ, Crossplane, Config Connector, etc)
3. At least one service instance in a `Namespace` called `service-instances`

The following chapters refer to the levels of service consumption as described in the documentation referenced above and gives detailed instructions on how to implement each with the setup we have.

In chapter, there will be instructions on steps to be taken in a developer `Namespace`. To follow these instructions, create such a `Namespace` first:

```
kubectl create ns --dry-run=client -o yaml test | kubectl apply -f -
kubectl label namespaces test apps.tanzu.vmware.com/tap-ns=""
```

## Level 2 - Resource Claims

In order to allow app teams to create a `ResourceClaim` in their developer `Namespace` that claims our service resource, we need to create a `ResourceClaimPolicy` which exposes the service resource to the developer `Namespace`. This way, app teams can claim that specific service instance.

![img](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/Images/images-stk-4-levels-2.png)

### Expose The Service

1. Create `ResourceClaimPolicy` that exposes all `Postgres` resources to all `Namespace`s 

    ```
    cat <<EOF | kubectl -n service-instances apply -f -
    apiVersion: services.apps.tanzu.vmware.com/v1alpha1
    kind: ResourceClaimPolicy
    metadata:
      name: postgres-to-all
    spec:
      subject:
        kind: Postgres
        group: sql.tanzu.vmware.com
        selector:
          matchLabels:
            app: postgres
      consumingNamespaces: [ "*" ]
    EOF
    ```

### Claim The Service

1. The the app team may claim the service:

    ```
    tanzu service resource-claim create petclinic-db \
      -n test \
      --resource-api-version sql.tanzu.vmware.com/v1 \
      --resource-kind Postgres \
      --resource-name pg-1 \
      --resource-namespace service-instances
    ```

2. Verify successful claim

    ```
    tanzu services resource-claims get petclinic-db --namespace test
    ```
    Expected output:
    ```
    Name: petclinic-db
    Status:
      Ready: True
    Namespace: test
    Claim Reference: services.apps.tanzu.vmware.com/v1alpha1:ResourceClaim:petclinic-db
    Resource Reference:
      Name: pg-1
      Namespace: service-instances
      Group: sql.tanzu.vmware.com
      Version: v1
      Kind: Postgres
    ```

3. Note that the `ResourceClaim` resource is service binding compatible by providing a `status.binding.name` field:

    ```
    kubectl -n test get resourceclaim petclinic-db -o jsonpath='{.status.binding.name}'
    ```

    Expected output:
    ```
    pg-1-app-user-db-secret
    ```

4. Note that the `Secret` with name `pg-1-app-user-db-secret` has been created in the developer `Namespace` and it's contents align with the service binding specification:

    ```
    kubectl -n test get secret pg-1-app-user-db-secret -o yaml | yq '.data'
    ```
    Expected output:
    ```
    database: ...
    host: ...
    password: ...
    port: ...
    provider: ...
    type: ...
    uri: ...
    username: ...
    ```
  
### Validation

Finally, let us validate this configuration by spinning up a service and bind it to out `ResourceClaim`. 

1. Create a `Workload` with a service reference

    The `--service-ref` parameter create the binding with the service by referencing the `ResourceClaim` created earlier.

    ```
    tanzu apps workload create petclinic -n test \
      -l "app.kubernetes.io/part-of=petclinic" \
      -l "apps.tanzu.vmware.com/workload-type=web" \
      -l "apps.tanzu.vmware.com/has-tests=true" \
      --build-env "BP_JVM_VERSION=17" \
      --git-repo https://github.com/spring-projects/spring-petclinic.git \
      --git-branch main \
      --service-ref "database=services.apps.tanzu.vmware.com/v1alpha1:ResourceClaim:petclinic-db"
    ```

2. Note that `tanzu apps workload get` now displays a "Services" section

    ```
    tanzu apps workload get petclinic --namespace test
    ```
    Expected output
    ```
    🔁 Services
      CLAIM      NAME           KIND            API VERSION
      database   petclinic-db   ResourceClaim   services.apps.tanzu.vmware.com/v1alpha1
    ```

### Cleanup

Run this cleanup step before you proceed to the next chapter.

```
tanzu -n test app workload delete test
tanzu -n test services resource-claims delete petclinic-db
```

## Level 3 - Class Claims And Pool-Based Classes

While in level 2, app teams claimed a specific service instances, they merely claim a `ClusterInstanceClass`. That class has a pool of service instances and the app team will receive a free service instance that matches their requirements. Instead of a `ResourceClaim` the app team creates a `ClassClaim`.

![img](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/Images/images-stk-4-levels-3.png)

1. Create the `ClusterInstanceClass`

    ```
    cat <<EOF | kubectl apply -f -
    apiVersion: services.apps.tanzu.vmware.com/v1alpha1
    kind: ClusterInstanceClass
    metadata:
      name: postgres
    spec:
      description:
        short: "PostgreSQL Databases"
      pool:
        group: sql.tanzu.vmware.com
        kind: Postgres
    EOF
    ```

2. Verify the discoverability of the `Postgres` service instance

    ```
    tanzu services claimable list --class postgres
    ```
    Expected output:
    ```
    NAME  NAMESPACE          KIND      APIVERSION
    pg-1  service-instances  Postgres  sql.tanzu.vmware.com/v1
    ```



## Level 4 - Dynamic Provisioning

While in level 3, the `ClusterInstanceClass` had an explicitly defined pool of **existing** service instances, level 4 introduces a provisioner in that place. The provisioner will dynamically provision service instances as claimed via the `ClusterInstanceClass`

![img](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/Images/images-stk-4-levels-4.png)