### Access TAP GUI

1. Open your browser (ideally, use Google Chrome as it allows you to accept the self-signed certificate)
2. Navigate to [https://tap-gui.$TAP_DOMAIN](http://tap-gui.$TAP_DOMAIN)
3. In the "Guest" panel, click "ENTER" to proceed as a guest user

### Deploy A Test Workload

1. Create a developer namespace
    ```bash
    kubectl create ns --dry-run=client -o yaml test | kubectl apply -f -
    kubectl label namespaces test apps.tanzu.vmware.com/tap-ns=""
    ```

2. Deploy the workload
    ```bash
    tanzu apps workload create petclinic -n test \
      -l "app.kubernetes.io/part-of=petclinic" \
      -l "apps.tanzu.vmware.com/workload-type=web" \
      --build-env "BP_JVM_VERSION=17" \
      --git-repo https://github.com/spring-projects/spring-petclinic.git \
      --git-branch main
    ```

3. Monitor progress

    Run the following command and check the readiness and health state of supply chain resources:
    ```bash
    tanzu apps workload get petclinic -n test
    ```

    Tail the logs of the supply chain execution:
    ```bash
    tanzu apps workload tail petclinic --namespace test
    ```

    Watch the progress in the UI:
    1. Open your browser at [https://tap-gui.DOMAIN](http://tap-gui.DOMAIN)
    2. In the menu, navigate to "Supply Chains"
    3. Click on `petclinic`

    &nbsp;
    
4. Access the application

    1. Run the following command to retrieve the application URL. The URL is displayed in the "Knative Services" section.
        ```bash
        tanzu apps workload get petclinic --namespace test
        ```

    2. Open the URL in your browser. You should see the Petclinic web UI.