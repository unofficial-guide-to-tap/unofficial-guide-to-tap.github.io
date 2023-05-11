# Create A Developer Service Account

In Kubernetes, a service account is a special type of account that is used by applications and processes. Unlike regular user accounts, service accounts are managed by the Kubernetes API and tied to specific namespaces within the cluster. You may want to use `ServiceAccount`s to developer `Namespace`s for the following situations:

* Application teams intend to connect to their developer `Namespace` with some form of automation like a CI/CD pipeline.
* You haven't connected TAP's Kubernetes cluster to any identity provider and don't intend to do so, yet users need access to their `Namespace`
* You want to test your platform and minimc the exact roles and permissions your users have.

## Setup The Service Account

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

4. Assign roles

    In this example, we're going to assign (bind) all the pre-defined roles for users that TAP ships.

    ```bash
    cat <<EOF | kubectl -n $NAMESPACE apply -f -
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: $USERNAME-app-editor
    subjects:
    - kind: ServiceAccount
      name: $USERNAME
    roleRef:
      kind: ClusterRole
      name: app-editor
      apiGroup: rbac.authorization.k8s.io
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: $USERNAME-app-operator
    subjects:
    - kind: ServiceAccount
      name: $USERNAME
    roleRef:
      kind: ClusterRole
      name: app-operator
      apiGroup: rbac.authorization.k8s.io
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: $USERNAME-app-viewer
    subjects:
    - kind: ServiceAccount
      name: $USERNAME
    roleRef:
      kind: ClusterRole
      name: app-viewer
      apiGroup: rbac.authorization.k8s.io
    EOF
    ```

5. Set up a kubeconfig context

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
