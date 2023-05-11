# Tanzu SQL for Kubernetes

## Parameters

In this section, we set up some environment variables that will be referenced down the line in the installation steps.

```bash
TANZUNET_USERNAME="..."
TANZUNET_PASSWORD="..."

INSTALL_REGISTRY_HOSTNAME="..."
INSTALL_REGISTRY_REPO="..."
INSTALL_REGISTRY_USERNAME="..."
INSTALL_REGISTRY_PASSWORD="..."
```

You may e.g. copy the code above, edit and execute it in your shell using `EDITOR=vim fc`. Alternatively, save the copy above to a file like `sql-params.sh` and load it into your shell with `source sql-params.sh`. The latter makes it easier to load them again after you have exited your shell.

- **TANZUNET_USERNAME**: The username to authenticate with Tanzu Network. 
- **TANZUNET_PASSWORD**: The password to authenticate with Tanzu Network. 
- **INSTALL_REGISTRY_HOSTNAME**: The hostname of your registry you use as a package mirror of Tanzu Network
- **INSTALL_REGISTRY_REPO**: The repository in that registry.
- **INSTALL_REGISTRY_USERNAME**: The username to authenticate with the registry. 
- **INSTALL_REGISTRY_PASSWORD**: The password to authenticate with the registry. 

> If you're using GCR on GCP, you will need to provide a service account key JSON as `INSTALL_REGISTRY_PASSWORD`. A quick way to convert the JSON to a string is `cat $HOME/key.json | jq -c | jq -R | sed 's/^"//' | sed 's/"$//'`.

## Installation

### Relocate Images

1. Docker login to Tanzu Network
    ```bash
    echo "$TANZUNET_PASSWORD" | \
    docker login registry.tanzu.vmware.com \
      -u $TANZUNET_USERNAME \
      --password-stdin
    ```

2. Docker login to your container registry
    ```bash
    echo "$INSTALL_REGISTRY_PASSWORD" | \
    docker login $INSTALL_REGISTRY_HOSTNAME \
      -u $INSTALL_REGISTRY_USERNAME \
      --password-stdin
    ```

    **NOTE:** Depending on your cloud platform, you might be using different ways to authenticate. Refer to the respective installation guide for details.

3. Mirror the packages

    ```bash
    imgpkg copy \
      -b registry.tanzu.vmware.com/packages-for-vmware-tanzu-data-services/tds-packages:1.7.0 \
      --to-repo $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REGISTRY_REPO/tds-packages
    ```

### Tanzu Postgres

#### Installation

1. Create the `Namespace` for the operator

    ```bash
    kubectl create ns postgres-tanzu-operator
    ```

2. Set up a `Secret` with credentials to the container registry containing the `Package`s

    ```bash
    kubectl create secret docker-registry regsecret \
      --namespace postgres-tanzu-operator \
      --docker-server=$INSTALL_REGISTRY_HOSTNAME \
      --docker-username="$INSTALL_REGISTRY_USERNAME" \
      --docker-password="$INSTALL_REGISTRY_PASSWORD"
    ```

3. Register the `PackageRepository`

    ```bash
    tanzu package repository add tanzu-data-services-repository \
      --url $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REGISTRY_REPO/tds-packages:1.7.0 \
      --namespace postgres-tanzu-operator
    ```

4. Install the `Package` for the Postgres operator

    ```bash
    cat <<EOF > postgres-operator.yaml
    dockerRegistrySecretName: regsecret
    EOF
    ```
    ```bash
    tanzu package install postgres-operator \
      --package postgres-operator.sql.tanzu.vmware.com \
      --version 2.0.1 \
      --namespace postgres-tanzu-operator \
      --values-file postgres-operator.yaml
    ```

#### Validation

1. Check availability of CRDs

    ```bash
    kubectl api-resources | grep postgres
    ```

    Expected output:
    ```
    postgres                          pg                                     sql.tanzu.vmware.com/v1                                             true         Postgres
    postgresbackuplocations                                                  sql.tanzu.vmware.com/v1                                             true         PostgresBackupLocation
    postgresbackups                                                          sql.tanzu.vmware.com/v1                                             true         PostgresBackup
    postgresbackupschedules                                                  sql.tanzu.vmware.com/v1                                             true         PostgresBackupSchedule
    postgresrestores                                                         sql.tanzu.vmware.com/v1                                             true         PostgresRestore
    postgresversions                                                         sql.tanzu.vmware.com/v1                                             false        PostgresVersion
    ```

2. Check available Postgres versions

    ```bash
    kubectl get postgresversions
    ```
    Expected output:
    ```
    NAME          DB VERSION
    postgres-11   11.19
    postgres-12   12.14
    postgres-13   13.10
    postgres-14   14.7
    postgres-15   15.2
    ```


### Tanzu MySQL

#### Installation

1. Create the `Namespace` for the operator

    ```bash
    kubectl create ns mysql-tanzu-operator
    ```

2. Set up a `Secret` with credentials to the container registry containing the `Package`s

    ```bash
    kubectl create secret docker-registry regsecret \
      --namespace mysql-tanzu-operator \
      --docker-server=$INSTALL_REGISTRY_HOSTNAME \
      --docker-username="$INSTALL_REGISTRY_USERNAME" \
      --docker-password="$INSTALL_REGISTRY_PASSWORD"
    ```

3. Register the `PackageRepository`

    ```bash
    tanzu package repository add tanzu-data-services-repository \
      --url $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REGISTRY_REPO/tds-packages:1.7.0 \
      --namespace mysql-tanzu-operator
    ```

4. Install the `Package` for the MySQL operator

    ```bash
    cat <<EOF > mysql-operator.yaml
    imagePullSecretName: regsecret
    EOF
    ```
    ```
    tanzu package install mysql-operator \
      --package mysql-operator.with.sql.tanzu.vmware.com \
      --version 1.7.0 \
      --namespace mysql-tanzu-operator \
      --values-file mysql-operator.yaml
    ```

#### Validation

1. Check availability of CRDs

    ```bash
    kubectl api-resources | grep mysql
    ```

    Expected output:
    ```
    mysqlbackuplocations                                                     with.sql.tanzu.vmware.com/v1                                        true         MySQLBackupLocation
    mysqlbackups                                                             with.sql.tanzu.vmware.com/v1                                        true         MySQLBackup
    mysqlbackupschedules                                                     with.sql.tanzu.vmware.com/v1                                        true         MySQLBackupSchedule
    mysqlrestores                                                            with.sql.tanzu.vmware.com/v1                                        true         MySQLRestore
    mysqls                                                                   with.sql.tanzu.vmware.com/v1                                        true         MySQL
    mysqlversions                                                            with.sql.tanzu.vmware.com/v1                                        false        MySQLVersion
    ```

2. Check available MySQL versions

    ```bash
    kubectl get mysqlversions
    ```
    Expected output:
    ```
    NAME           DB VERSION
    mysql-8.0.27   8.0.27
    mysql-8.0.28   8.0.28
    mysql-8.0.29   8.0.29
    mysql-8.0.31   8.0.31
    mysql-latest   8.0.3
    ```

2. Create a database instance: Check [usage](#usage) section for examples

## Usage Examples

```bash
kubectl create namespace service-instances
```

```bash
kubectl create secret docker-registry regsecret \
  --namespace service-instances \
  --docker-server=$INSTALL_REGISTRY_HOSTNAME \
  --docker-username="$INSTALL_REGISTRY_USERNAME" \
  --docker-password="$INSTALL_REGISTRY_PASSWORD"
```

### Postgres

```bash
cat <<EOF | kubectl -n service-instances apply -f -
apiVersion: sql.tanzu.vmware.com/v1
kind: Postgres
metadata:
  name: pg-1
spec:
  storageSize: 1G
  highAvailability:
    enabled: false
EOF
```

```bash
kubectl -n service-instances get postgres,pods,services
```

Expected output:
```
NAME                                 STATUS    DB VERSION   BACKUP LOCATION   AGE
postgres.sql.tanzu.vmware.com/pg-1   Running   15.2                           53m

NAME                 READY   STATUS    RESTARTS   AGE
pod/pg-1-0           5/5     Running   0          53m
pod/pg-1-monitor-0   4/4     Running   0          50m

NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/pg-1         ClusterIP   10.3.253.45   <none>        5432/TCP   53m
service/pg-1-agent   ClusterIP   None          <none>        <none>     53m
```

### MySQL

```bash
cat <<EOF | kubectl -n service-instances apply -f -
apiVersion: with.sql.tanzu.vmware.com/v1
kind: MySQL
metadata:
  name: mysql-1
spec:
  storageSize: 1G
  highAvailability:
    enabled: false
EOF
```

```bash
kubectl -n service-instances get mysql,pods,services
```

Expected output:
```
NAME                                           READY   STATUS    AGE   OPERATOR VERSION   DB VERSION   UPDATE STATUS      USER ACTION
mysql.with.sql.tanzu.vmware.com/mysql-sample   true    Running   66s   1.7.0              8.0.31       NoUpdateRequired

NAME                 READY   STATUS    RESTARTS   AGE
pod/mysql-sample-0   3/3     Running   0          65s

NAME                           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)              AGE
service/mysql-sample           ClusterIP   10.3.248.124   <none>        3306/TCP,33060/TCP   65s
service/mysql-sample-members   ClusterIP   None           <none>        3306/TCP,33060/TCP   65s
```
