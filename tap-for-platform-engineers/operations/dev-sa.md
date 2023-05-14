# Create A Developer Service Account

In Kubernetes, a service account is a special type of account that is used by applications and processes. Unlike regular user accounts, service accounts are managed by the Kubernetes API and tied to specific namespaces within the cluster. You may want to use `ServiceAccount`s to developer `Namespace`s for the following situations:

* Application teams intend to connect to their developer `Namespace` with some form of automation like a CI/CD pipeline.
* You haven't connected TAP's Kubernetes cluster to any identity provider and don't intend to do so, yet users need access to their `Namespace`
* You want to test your platform and minimc the exact roles and permissions your users have.

## Create The Service Account

```
USERNAME="dev-sa"
NAMESPACE="test"
```

1. Create a developer `Namespace`

    ```bash
    kubectl create ns --dry-run=client -o yaml $NAMESPACE | kubectl apply -f -
    kubectl label namespaces $NAMESPACE apps.tanzu.vmware.com/tap-ns=""
    ```

2. Create the `ServiceAccount`

    ```bash
    cat <<EOF | kubectl -n $NAMESPACE apply -f -
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: $USERNAME
    EOF
    ```

3. Create an authentication `Secret`

    ```bash
    cat <<EOF | kubectl -n $NAMESPACE apply -f -
    apiVersion: v1
    kind: Secret
    type: kubernetes.io/service-account-token
    metadata:
      name: $USERNAME-token
      annotations:
        kubernetes.io/service-account.name: $USERNAME
    EOF
    ```

## Configure RBAC

With a regular `User` this step would be very easy with the `rbac` plugin for Tanzu CLI. You would simple run something like

```bash
for I in editor operator viewer; do
  tanzu rbac binding add --user dev1 --role app-$I --namespace test
done
```

and the CLI would do the following for you:

- Create a `RoleBinding` called "app-editor" in the `Namespace` "test" and add the `User` "dev1" to it
- Create a `RoleBinding` called "app-operator" in the `Namespace` "test" and add the `User` "dev1" to it
- Create a `RoleBinding` called "app-viewer"in the `Namespace` "test" and add the `User` "dev1" to it
- Add the `User` "dev1" to the `ClusterRoleBinding` "app-editor-cluster-access"
- Add the `User` "dev1" to the `ClusterRoleBinding` "app-operator-cluster-access"
- Add the `User` "dev1" to the `ClusterRoleBinding` "app-viewer-cluster-access"

The output would look something like this:
```
Created RoleBinding 'app-editor' in namespace 'test'
Added User 'dev-sa' to RoleBinding 'app-editor' in namespace 'test'
Added User 'dev-sa' to ClusterRoleBinding 'app-editor-cluster-access'
Created RoleBinding 'app-operator' in namespace 'test'
Added User 'dev-sa' to RoleBinding 'app-operator' in namespace 'test'
Added User 'dev-sa' to ClusterRoleBinding 'app-operator-cluster-access'
Created RoleBinding 'app-viewer' in namespace 'test'
Added User 'dev-sa' to RoleBinding 'app-viewer' in namespace 'test'
Added User 'dev-sa' to ClusterRoleBinding 'app-viewer-cluster-access'
```

Since we're using a `ServiceAccount` and because Tanzu CLI's RBAC plugin does not support that, we need to perform the same steps manually. Save the following script as `sa-rbac.sh`:

```bash
#!/bin/bash
set -euo pipefail

NAMESPACE="$1"
SA_NAME="$2"

# Create RoleBindings in Namespace

for I in editor operator viewer; do
  cat <<EOF | kubectl apply -n $NAMESPACE -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-$I
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: app-$I
EOF
done

# Create Patch for (Cluster)RoleBindings
PATCH_FILE="$(mktemp)"
cat <<EOF > $PATCH_FILE
[{
  "op": "add",
  "path": "/subjects/-",
  "value": {
    "kind": "ServiceAccount",
    "name": "$SA_NAME",
    "namespace": "$NAMESPACE"
  }
}]
EOF

# Add ServiceAccount to RoleBindings
for I in editor operator viewer; do
  kubectl -n $NAMESPACE patch rolebinding app-$I --type json --patch-file $PATCH_FILE
done

# Add ServiceAccount to ClusterRoleBindings
for I in editor operator viewer; do
  kubectl -n $NAMESPACE patch clusterrolebinding app-$I-cluster-access --type json --patch-file $PATCH_FILE
done
```

Then run: 
```bash
bash sa-rbac.sh test dev-sa
```

> Please not that the script does not check if the `ServiceAccount` is already in the list of `subjects` of the `(Cluster)RoleBinding`. Instead the `ServiceAccount` will be appended every time the script runs. While this is not a problem for RBAC it pollutes your `(Cluster)RoleBindings` over time.


## Create A Kubeconfig Context

1. Set up a kubeconfig context

  Extract the token
  ```bash
  TOKEN="$(kubectl -n $NAMESPACE get secrets $USERNAME-token -o jsonpath='{.data.token}' | base64 -d)"
  ```

  Store the CA cert to a file
  ```bash
  CA_CRT_FILE="$(mktemp)"
  kubectl -n $NAMESPACE get secrets $USERNAME-token -o jsonpath='{.data.ca\.crt}' | base64 -d > $CA_CRT_FILE
  ```

  Create cluster, user and context in your kubeconf
  ```bash
  CONTEXT_NAME="tap"
  CLUSTER_NAME="tap-cluster"
  K8S_API="..." # e.g. https://1.2.3.4
  ```

  ```bash
  kubectl config set-cluster $CLUSTER_NAME --server=$K8S_API --certificate-authority=$CA_CRT_FILE --embed-certs=true
  kubectl config set-context $CONTEXT_NAME --cluster=$CLUSTER_NAME
  kubectl config set-credentials $USERNAME --token="$TOKEN"
  kubectl config set-context $CONTEXT_NAME --user=$USERNAME
  ```

## Validation

1. Switch the context to the `dev-sa` service account

  ```bash
  kubectl config use-context $CONTEXT_NAME
  ```

2. List all `Pods` in the `test` `Namespace`:

    ```bash
    kubectl -n test get pods
    ```

    Expected output
    ```
    No resources found in test namespace
    ```

3. List all  `Namespaces`

    ```bash
    kubectl get namespaces
    ```

    Expected output
    ```
    Error from server (Forbidden): namespaces is forbidden: User "system:serviceaccount:test:dev-sa" cannot list resource "namespaces" in API group "" at the cluster scope
    ```

4. Deploy a test `Workload`

    ```bash
    tanzu apps workload create petclinic -n $NAMESPACE \
      -l "app.kubernetes.io/part-of=petclinic" \
      -l "apps.tanzu.vmware.com/workload-type=web" \
      -l "apps.tanzu.vmware.com/has-tests=true" \ # Only required if you installed the testing and scanning ootb supply chain
      --build-env "BP_JVM_VERSION=17" \
      --git-repo https://github.com/spring-projects/spring-petclinic.git \
      --git-branch main
    ```

5. Observe progress

    ```bash
    tanzu apps workload get petclinic --namespace $NAMESPACE
    ```
