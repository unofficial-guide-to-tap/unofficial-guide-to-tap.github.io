# OOTB Supply Chain With Testing

## Optional: Validate Installed Supply Chains

1. Validate the `Package` "ootb-supply-chain-basic" is installed:

    ```
    tanzu -n tap-install package installed list | grep supply
    ```
    Expected output:
    ```
    ootb-supply-chain-basic   ootb-supply-chain-basic.tanzu.vmware.com      0.11.3            Reconcile succeeded
    ```

2. List available `ClusterSupplyChain`s in the cluster:
    ```
    kubectl get clustersupplychains
    ```
    Expected output:
    ```
    NAME                 READY   REASON   AGE
    basic-image-to-url   True    Ready    4d18h
    source-to-url        True    Ready    4d18h
    ```

## Install The Supply Chain With Testing

1. Make the following changes to your `values.yaml`: 

    ```
    supply_chain: testing
    ```

2. Update your TAP deployment

    ```
    tanzu package installed update tap \
      -n "tap-install" \
      -p tap.tanzu.vmware.com \
      -v "1.4.4" \
      --values-file values.yaml \
      --wait="false"
    ```

## Validation 

### Validate Successful Installation

```
kubectl get clustersupplychains
```
Expected output:
```
NAME                   READY   REASON   AGE
source-test-to-url     True    Ready    23s
testing-image-to-url   True    Ready    23s
```

### Run A Workload With Testing

1. Create a developer namespace
    ```
    kubectl create ns --dry-run=client -o yaml test | kubectl apply -f -
    kubectl label namespaces test apps.tanzu.vmware.com/tap-ns=""
    ```

1. Create a compatible Tekton `Pipeline`

    ```
    cat <<EOF | kubectl -n test apply -f -
    apiVersion: tekton.dev/v1beta1
    kind: Pipeline
    metadata:
      name: noop-pipeline
      labels:
        apps.tanzu.vmware.com/pipeline: test     # (!) required
    spec:
      params:
        - name: source-url                       # (!) required
        - name: source-revision                  # (!) required
      tasks:
        - name: test
          params:
            - name: source-url
              value: \$(params.source-url)
            - name: source-revision
              value: \$(params.source-revision)
          taskSpec:
            params:
              - name: source-url
              - name: source-revision
            steps:
              - name: test
                image: gradle
                script: |-
                  find /
                  gradle test
    EOF
    ```

3. Create a `Workload` to be tested

    ```
    tanzu apps workload create petclinic -n test \
      -l "app.kubernetes.io/part-of=petclinic" \
      -l "apps.tanzu.vmware.com/workload-type=web" \
      -l "apps.tanzu.vmware.com/has-tests=true" \
      --build-env "BP_JVM_VERSION=17" \
      --git-repo https://github.com/spring-projects/spring-petclinic.git \
      --git-branch main
    ```

4. Watch the progress

    Did a `PipelineRun` get created?
    ```
    kubectl -n test get pipelinerun
    ```
    Expected output:
    ```
    NAME              SUCCEEDED   REASON    STARTTIME   COMPLETIONTIME
    petclinic-89vwv   Unknown     Running   11s
    ```

    Did a `Pod` get started to execute the tests?
    ```
    kubectl -n test get pods
    ```
    Expected output:
    ```
    NAME                          READY   STATUS      RESTARTS   AGE
    petclinic-89vwv-test-pod      0/1     Completed   0          38s
    petclinic-build-1-build-pod   0/1     Init:0/6    0          11s
    ```

    What's the log output of the test?
    ```
    kubectl -n test logs -f petclinic-89vwv-test-pod
    ```
    Expected output:
    ```
    Defaulted container "step-test" out of: step-test, prepare (init), place-scripts (init)
    Didnu nuffin. LOL!
    ```