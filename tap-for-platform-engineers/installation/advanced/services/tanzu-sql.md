# Tanzu SQL for Kubernetes

1. [Installation](#installation)
    1. [Relocate Images](#relocate-images)
    2. [Tanzu Postgres](#install-tanzu-postgres)
    3. [Tanzu MySQL](#install-tanzu-mysql)
2. [Usage Examples](#usage-examples)
    1. [Postgres](#create-a-postgres-database)
    2. [MySQL](#create-a-mysql-database)
---

Tanzu SQL for Kubernetes provides Kubernetes operators for Postgres and MySQL. In this guide, you will install both operators and create an instance of each service.

## Installation

Make sure set up the folloring variables before proceeding with this guide:

```
TDS_VERSION="1.7.0"
TANZUNET_USERNAME="..."
TANZUNET_PASSWORD="..."

POSTGRES_VERSION="2.0.1"
MYSQL_VERSION="1.7.0"

INSTALL_REGISTRY_HOSTNAME="..."
INSTALL_REGISTRY_REPO="..."
INSTALL_REGISTRY_USERNAME="..."
INSTALL_REGISTRY_PASSWORD="..."
```
- **TDS_VERSION**: The version of Tanzu Data Services as available in Tanzu Network
- **TANZUNET_USERNAME**: The username to authenticate with Tanzu Network. 
- **TANZUNET_PASSWORD**: The password to authenticate with Tanzu Network. 
- **POSTGRES_VERSION**: The version of the Postgres operator to install
- **MYSQL_VERSION**: The version of the MySQL operator to install
- **INSTALL_REGISTRY_HOSTNAME**: The hostname of your registry you use as a package mirror of Tanzu Network
- **INSTALL_REGISTRY_REPO**: The repository in that registry.
- **INSTALL_REGISTRY_USERNAME**: The username to authenticate with the registry. 
- **INSTALL_REGISTRY_PASSWORD**: The password to authenticate with the registry. 


### Relocate Images

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

    **NOTE:** Depending on your cloud platform, you might be using different ways to authenticate. Refer to the respective installation guide for details.

3. Mirror the packages

    ```
    imgpkg copy \
      -b registry.tanzu.vmware.com/packages-for-vmware-tanzu-data-services/tds-packages:$TDS_VERSION \
      --to-repo $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REGISTRY_REPO/tds-packages
    ```

### Tanzu Postgres

#### Installation

1. Create the `Namespace` for the operator

    ```
    kubectl create ns postgres-tanzu-operator
    ```

2. Set up a `Secret` with credentials to the container registry containing the `Package`s

    ```
    kubectl create secret docker-registry regsecret \
      --namespace postgres-tanzu-operator \
      --docker-server=$INSTALL_REGISTRY_HOSTNAME \
      --docker-username="$INSTALL_REGISTRY_USERNAME" \
      --docker-password="$INSTALL_REGISTRY_PASSWORD"
    ```

3. Register the `PackageRepository`

    ```
    tanzu package repository add tanzu-data-services-repository \
      --url $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REGISTRY_REPO/tds-packages:$TDS_VERSION \
      --namespace postgres-tanzu-operator
    ```

4. Install the `Package` for the Postgres operator

    ```
    cat <<EOF > postgres-operator.yaml
    dockerRegistrySecretName: regsecret
    EOF
    ```
    ```
    tanzu package install postgres-operator \
      --package-name postgres-operator.sql.tanzu.vmware.com \
      --version $POSTGRES_VERSION \
      --namespace postgres-tanzu-operator \
      -f postgres-operator.yaml
    ```

#### Validation

1. Check availability of CRDs

    ```
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

    ```
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

    ```
    kubectl create ns mysql-tanzu-operator
    ```

2. Set up a `Secret` with credentials to the container registry containing the `Package`s

    ```
    kubectl create secret docker-registry regsecret \
      --namespace mysql-tanzu-operator \
      --docker-server=$INSTALL_REGISTRY_HOSTNAME \
      --docker-username="$INSTALL_REGISTRY_USERNAME" \
      --docker-password="$INSTALL_REGISTRY_PASSWORD"
    ```

3. Register the `PackageRepository`

    ```
    tanzu package repository add tanzu-data-services-repository \
      --url $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REGISTRY_REPO/tds-packages:$TDS_VERSION \
      --namespace mysql-tanzu-operator
    ```

4. Install the `Package` for the MySQL operator

    ```
    cat <<EOF > mysql-operator.yaml
    imagePullSecretName: regsecret
    EOF
    ```
    ```
    tanzu package install mysql-operator \
      --package-name mysql-operator.with.sql.tanzu.vmware.com \
      --version $MYSQL_VERSION \
      --namespace mysql-tanzu-operator \
      -f mysql-operator.yaml
    ```

#### Validation

1. Check availability of CRDs

    ```
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

    ```
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

```
kubectl create namespace service-instances
```

```
kubectl create secret docker-registry regsecret \
  --namespace service-instances \
  --docker-server=$INSTALL_REGISTRY_HOSTNAME \
  --docker-username="$INSTALL_REGISTRY_USERNAME" \
  --docker-password="$INSTALL_REGISTRY_PASSWORD"
```

### Postgres

```
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

```
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

```
cat <<EOF | kubectl -n service-instances apply -f -
apiVersion: with.sql.tanzu.vmware.com/v1
kind: MySQL
metadata:
  name: mysql-sample
spec:
  storageSize: 1G
  highAvailability:
    enabled: false
EOF
```

```
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

---
Next: [Services Toolkit](./services-toolkit.md)