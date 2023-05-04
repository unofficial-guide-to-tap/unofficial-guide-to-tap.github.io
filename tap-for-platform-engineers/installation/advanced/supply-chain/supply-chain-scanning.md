# OOTB Supply Chain With Testing And Scanning

## Install The Supply Chain With Testing And Scanning

1. Update `values.yaml`

    1. Change the Supply Chain `Package`

        Instead of the default value of `basic`, we use `testing_scanning` to tell the "tap" `Package` to install the `Package` called "ootb-supply-chain-testing-scanning" for the OOTB supply chain. This will delete any other ootb supply chain `Package`.

        ```yaml
        supply_chain: testing_scanning
        ````

    2. Export the Metadata Store certificate to all `Namespace`s

        This tells the `Package` that installs the Metadata Store service, to create a `SecretExport` of the certificate to all other namespaces in the cluster. This way, our developer namespaces will have an `app-tls-cert` which allows Grype to connect and push scan results to the Metadata Store.


        ```yaml
        metadata_store:
          ns_for_export_app_cert: "*"
        ```

    3. Setup the TAP GUI proxy to access the Metadata Store

      In order to see scan results in TAP GUI, the UI will make a call to `/api/proxy/metadata-store/sources/vulnerabilities` which is a proxy that needs to be setup to forward this request to the Metadata Store service. This proxy needs to be configured manually. As Metadata Store authorizes any incoming request via bearer token, TAP GUI needs to have that token.

      ```bash
      kubectl get secrets metadata-store-read-write-client -n metadata-store \
        -o jsonpath="{.data.token}" | base64 -d
      ```

      ```yaml
      tap_gui:
        app_config:
          proxy:
            /metadata-store:
              target: https://metadata-store-app.metadata-store:8443/api/v1
              changeOrigin: true
              secure: false
              headers:
                Authorization: "Bearer METADATA_STORE_READ_WRITE_TOKEN"
                X-Custom-Source: project-star
      ```

2. Update your TAP deployment

    ```bash
    tanzu package installed update tap \
      -n "tap-install" \
      -p tap.tanzu.vmware.com \
      -v "1.5.0" \
      --values-file values.yaml \
      --wait="false"
    ```

3. Wait for all `Package`s to reconcile

    ```bash
    tanzu -n tap-install package installed list
    ```

## Validation 

### Validate Successful Installation

```bash
kubectl get clustersupplychains
```
Expected output:
```
NAME                         READY   REASON   AGE
scanning-image-scan-to-url   True    Ready    5m3s
source-test-scan-to-url      True    Ready    5m3s
```

### Run A Workload With Testing

1. Create a developer namespace
    ```bash
    kubectl create ns --dry-run=client -o yaml test | kubectl apply -f -
    kubectl label namespaces test apps.tanzu.vmware.com/tap-ns=""
    ```

1. Create a compatible Tekton `Pipeline`

    ```bash
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
                  echo "Dindu nuffin."
                  exit 0
    EOF
    ```

2. Install a `ScanPolicy`

    The following `ScanPolicy` will allow allow through pretty much all vulnerabilites found. Change the value for `notAllowedSeverities` for example to `["Critical", "High", "UnknownSeverity"]` to be more restrictive.
    
    **Note:** With a more restrictive `ScanPolicy` the code deloyed in the next step (spring-petclinic) will fail that step.

    ```bash
    cat <<EOF | kubectl -n test apply -f -
    apiVersion: scanning.apps.tanzu.vmware.com/v1beta1
    kind: ScanPolicy
    metadata:
      name: scan-policy
      labels:
        'app.kubernetes.io/part-of': 'enable-in-gui'
    spec:
      regoFile: |
        package main

        # Accepted Values: "Critical", "High", "Medium", "Low", "Negligible", "UnknownSeverity"
        notAllowedSeverities := ["UnknownSeverity"]
        ignoreCves := []

        contains(array, elem) = true {
          array[_] = elem
        } else = false { true }

        isSafe(match) {
          severities := { e | e := match.ratings.rating.severity } | { e | e := match.ratings.rating[_].severity }
          some i
          fails := contains(notAllowedSeverities, severities[i])
          not fails
        }

        isSafe(match) {
          ignore := contains(ignoreCves, match.id)
          ignore
        }

        deny[msg] {
          comps := { e | e := input.bom.components.component } | { e | e := input.bom.components.component[_] }
          some i
          comp := comps[i]
          vulns := { e | e := comp.vulnerabilities.vulnerability } | { e | e := comp.vulnerabilities.vulnerability[_] }
          some j
          vuln := vulns[j]
          ratings := { e | e := vuln.ratings.rating.severity } | { e | e := vuln.ratings.rating[_].severity }
          not isSafe(vuln)
          msg = sprintf("CVE %s %s %s", [comp.name, vuln.id, ratings])
        }
    EOF
    ```

3. Create a `Workload` to be tested and scanned

    ```bash
    tanzu apps workload create petclinic -n test \
      -l "app.kubernetes.io/part-of=petclinic" \
      -l "apps.tanzu.vmware.com/workload-type=web" \
      -l "apps.tanzu.vmware.com/has-tests=true" \
      --build-env "BP_JVM_VERSION=17" \
      --git-repo https://github.com/spring-projects/spring-petclinic.git \
      --git-branch main
    ```

4. Watch the progress

    Run the following command and wait for the scan `Pod` to complete.

    ```bash
    tanzu apps workload get petclinic --namespace test
    ```

5. View the results

    1. Open TAP GUI in your browser
    2. Click the `Workloads` menu button on the left
    3. Click `petclinic` to open your `Workload`
    4. In the supply chain illustration click `Source Scanner`
    5. Scrolling down, you should see a list of vulnerabilities

    </br>
    
    ![scan results](scan-results.png)