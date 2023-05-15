# Dynamic Provisioning With Tanzu SQL Postgres

If you haven't already done so, proceed to [Tanzu SQL](tap-for-platform-engineers/installation/advanced/services/tanzu-sql.md) and follow the instructions to install Tanzu SQL.


## Setup The `Namespace`

1. Create a `Namespace` for `Postgres` resources

  ```
  kubectl create namespace tanzu-psql-service-instances
  ```

2. Create the `Secret` to the registry holding postgres images as described in the [Tanzu SQL](tap-for-platform-engineers/installation/advanced/services/tanzu-sql.md) installation guide.

  ```
  kubectl create secret docker-registry regsecret \
    --namespace tanzu-rabbitmq-operator \
    --docker-server=$INSTALL_REGISTRY_HOSTNAME \
    --docker-username="$INSTALL_REGISTRY_USERNAME"  \
    --docker-password="$INSTALL_REGISTRY_PASSWORD"
  ```

## Configure Crossplane

1. Create a `CompositeResourceDefinition`

    ```
    cat <<EOF | kubectl apply -f -
    # xpostgresqlinstances.database.tanzu.example.org.xrd.yaml

    ---
    apiVersion: apiextensions.crossplane.io/v1
    kind: CompositeResourceDefinition
    metadata:
      name: xpostgresqlinstances.database.tanzu.example.org
    spec:
      connectionSecretKeys:
      - provider
      - type
      - database
      - host
      - password
      - port
      - uri
      - username
      group: database.tanzu.example.org
      names:
        kind: XPostgreSQLInstance
        plural: xpostgresqlinstances
      versions:
      - name: v1alpha1
        referenceable: true
        schema:
          openAPIV3Schema:
            properties:
              spec:
              properties:
                storageGB:
                  type: integer
                  default: 20
              type: object
            type: object
        served: true
    EOF
    ```

2. Create a `Composition`

    ``` 
    cat <<EOF | kubectl apply -f -
    # xpostgresqlinstances.database.tanzu.example.org.composition.yaml

    ---
    apiVersion: apiextensions.crossplane.io/v1
    kind: Composition
    metadata:
      name: xpostgresqlinstances.database.tanzu.example.org
    spec:
      compositeTypeRef:
        apiVersion: database.tanzu.example.org/v1alpha1
        kind: XPostgreSQLInstance
      publishConnectionDetailsWithStoreConfigRef:
        name: default
      resources:
      - base:
          apiVersion: kubernetes.crossplane.io/v1alpha1
          kind: Object
          spec:
            forProvider:
              manifest:
                apiVersion: sql.tanzu.vmware.com/v1
                kind: Postgres
                metadata:
                  name: PATCHED
                  namespace: tanzu-psql-service-instances
                spec:
                  storageSize: 2G
            connectionDetails:
            - apiVersion: v1
              kind: Secret
              namespace: tanzu-psql-service-instances
              fieldPath: data.provider
              toConnectionSecretKey: provider
            - apiVersion: v1
              kind: Secret
              namespace: tanzu-psql-service-instances
              fieldPath: data.type
              toConnectionSecretKey: type
            - apiVersion: v1
              kind: Secret
              namespace: tanzu-psql-service-instances
              fieldPath: data.host
              toConnectionSecretKey: host
            - apiVersion: v1
              kind: Secret
              namespace: tanzu-psql-service-instances
              fieldPath: data.port
              toConnectionSecretKey: port
            - apiVersion: v1
              kind: Secret
              namespace: tanzu-psql-service-instances
              fieldPath: data.username
              toConnectionSecretKey: username
            - apiVersion: v1
              kind: Secret
              namespace: tanzu-psql-service-instances
              fieldPath: data.password
              toConnectionSecretKey: password
            - apiVersion: v1
              kind: Secret
              namespace: tanzu-psql-service-instances
              fieldPath: data.database
              toConnectionSecretKey: database
            - apiVersion: v1
              kind: Secret
              namespace: tanzu-psql-service-instances
              fieldPath: data.uri
              toConnectionSecretKey: uri
            writeConnectionSecretToRef:
              namespace: tanzu-psql-service-instances
        connectionDetails:
        - fromConnectionSecretKey: provider
        - fromConnectionSecretKey: type
        - fromConnectionSecretKey: host
        - fromConnectionSecretKey: port
        - fromConnectionSecretKey: username
        - fromConnectionSecretKey: password
        - fromConnectionSecretKey: database
        - fromConnectionSecretKey: uri
        patches:
          - fromFieldPath: metadata.name
            toFieldPath: spec.forProvider.manifest.metadata.name
            type: FromCompositeFieldPath
          - fromFieldPath: spec.storageSize
            toFieldPath: spec.forProvider.manifest.spec.persistence.storage
            transforms:
            - string:
                fmt: '%dG'
                type: Format
              type: string
            type: FromCompositeFieldPath
          - fromFieldPath: metadata.name
            toFieldPath: spec.writeConnectionSecretToRef.name
            transforms:
            - string:
                fmt: '%s-psql'
                type: Format
              type: string
            type: FromCompositeFieldPath
          - fromFieldPath: metadata.name
            toFieldPath: spec.connectionDetails[0].name
            transforms:
            - string:
                fmt: '%s-app-user-db-secret'
                type: Format
              type: string
            type: FromCompositeFieldPath
          - fromFieldPath: metadata.name
            toFieldPath: spec.connectionDetails[1].name
            transforms:
            - string:
                fmt: '%s-app-user-db-secret'
                type: Format
              type: string
            type: FromCompositeFieldPath
          - fromFieldPath: metadata.name
            toFieldPath: spec.connectionDetails[2].name
            transforms:
            - string:
                fmt: '%s-app-user-db-secret'
                type: Format
              type: string
            type: FromCompositeFieldPath
          - fromFieldPath: metadata.name
            toFieldPath: spec.connectionDetails[3].name
            transforms:
            - string:
                fmt: '%s-app-user-db-secret'
                type: Format
              type: string
            type: FromCompositeFieldPath
          - fromFieldPath: metadata.name
            toFieldPath: spec.connectionDetails[4].name
            transforms:
            - string:
                fmt: '%s-app-user-db-secret'
                type: Format
              type: string
            type: FromCompositeFieldPath
          - fromFieldPath: metadata.name
            toFieldPath: spec.connectionDetails[5].name
            transforms:
            - string:
                fmt: '%s-app-user-db-secret'
                type: Format
              type: string
            type: FromCompositeFieldPath
          - fromFieldPath: metadata.name
            toFieldPath: spec.connectionDetails[6].name
            transforms:
            - string:
                fmt: '%s-app-user-db-secret'
                type: Format
              type: string
            type: FromCompositeFieldPath
          - fromFieldPath: metadata.name
            toFieldPath: spec.connectionDetails[7].name
            transforms:
            - string:
                fmt: '%s-app-user-db-secret'
                type: Format
              type: string
            type: FromCompositeFieldPath
        readinessChecks:
          - type: MatchString
            fieldPath: status.atProvider.manifest.status.currentState
            matchString: "Running"
    EOF
    ```

3. Allow Crossplane to manage `Postgres` instances

    ```
    cat <<EOF | kubectl apply -f -
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: tanzu-postgres-read-writer
      labels:
        services.tanzu.vmware.com/aggregate-to-provider-kubernetes: "true"
    rules:
    - apiGroups:
      - sql.tanzu.vmware.com
      resources:
      - postgres
      verbs:
      - "*"
    EOF
    ```

## Configure Services Toolkit

1. Create the `ClusterInstanceClass`

    ```
    cat <<EOF | kubectl apply -f -
    ---
    apiVersion: services.apps.tanzu.vmware.com/v1alpha1
    kind: ClusterInstanceClass
    metadata:
      name: tanzu-psql
    spec:
      description:
        short: VMware Tanzu Postgres
      provisioner:
        crossplane:
          compositeResourceDefinition: xpostgresqlinstances.database.tanzu.example.org
    EOF
    ```

2. Allow users with the `app-operator-cluster-access` role to use the service class

    ```
    cat <<EOF | kubectl apply -f -
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: app-operator-claim-tanzu-psql
      labels:
        apps.tanzu.vmware.com/aggregate-to-app-operator-cluster-access: "true"
    rules:
    - apiGroups:
      - "services.apps.tanzu.vmware.com"
      resources:
      - clusterinstanceclasses
      resourceNames:
      - tanzu-psql
      verbs:
      - claim
    EOF
    ```

## Validate

As a developer:

1. Create a `ClassClaim` for the `ClusterInstanceClass`

  ```bash
  tanzu service class-claim create tanzu-psql-1 --class tanzu-psql -p storageGB=5 -n test
  ```

2. Validate the `ClassClaim` has ben created successfully

  ```
  tanzu services class-claims get tanzu-psql-1 --namespace test
  ```

As a platform engineer:

1. Check if the instance of `XPostgreSQLInstance` has been created

  ```
  kubectl get xpostgresqlinstances.database.tanzu.example.org -A
  ```
  Expected output:
  ```
  NAME                 SYNCED   READY   COMPOSITION                                       AGE
  tanzu-psql-1-jj2vg   True     False   xpostgresqlinstances.database.tanzu.example.org   101s
  ```

2. Has the `Postgres` instance been created in the `tanzu-psql-service-instances` `Namespace`?

  ```bash
  kubectl -n tanzu-psql-service-instances get postgres
  ```
  Expected output:
  ```
  NAME                 STATUS    DB VERSION   BACKUP LOCATION   AGE
  tanzu-psql-1-jj2vg   Running   15.2                           2m56s
  ```

As a developer:

1. Wait for `tanzu services class-claims get tanzu-psql-1 --namespace test` to show `Ready: True`

2. Check the `Secret` mentioned in the output of step 1 under `Status.Claimed Resource.Name` for example with `kubectl view-secret` or `kubectl get secret -o yaml`:

  ```
  kubectl view-secret -n test 450836cc-a4bf-4988-aa6c-5bb2e128e079 -a
  ```
  Expected output:
  ```
  database=tanzu-psql-1-jj2vg
  host=tanzu-psql-1-jj2vg.tanzu-psql-service-instances
  password=**********
  port=5432
  provider=vmware
  type=postgresql
  uri=postgresql://pgappuser:**********@tanzu-psql-1-jj2vg.tanzu-psql-service-instances:5432/tanzu-psql-1-jj2vg
  username=pgappuser
  ```

  Or

  ```
  kubectl -n test get secret 450836cc-a4bf-4988-aa6c-5bb2e128e079 -o yaml | yq .data
  ```
  Expected output: (values are base64 encoded)
  ```
  database: dGFuenUtcHNxbC0xLWpqMnZn
  host: dGFuenUtcHNxbC0xLWpqMnZnLnRhbnp1LXBzcWwtc2VydmljZS1pbnN0YW5jZXM=
  password: KioqKioqKioqKgo=
  port: NTQzMg==
  provider: dm13YXJl
  type: cG9zdGdyZXNxbA==
  uri: cG9zdGdyZXNxbDovL3BnYXBwdXNlcjoqKioqKioqKioqQHRhbnp1LXBzcWwtMS1qajJ2Zy50YW56dS1wc3FsLXNlcnZpY2UtaW5zdGFuY2VzOjU0MzIvdGFuenUtcHNxbC0xLWpqMnZnCg==
  username: cGdhcHB1c2Vy
  ```

<!-- https://vrelevant.net/crossplane-troubleshooting/ -->