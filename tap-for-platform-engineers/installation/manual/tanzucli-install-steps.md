1. Extract the downloaded archive
    ```bash
    mkdir tanzu-framework
    tar xvf tanzu-framework-linux-amd64-v0.28.1.1.tar -C tanzu-framework
    cd tanzu-framework
    ```

2. Install the executable
    ```bash
    sudo install cli/core/v0.28.1/tanzu-core-linux_amd64 /usr/local/bin/tanzu
    ```

3. Install the shipped plugins
    ```bash
    export TANZU_CLI_NO_INIT=true
    tanzu plugin install --local cli all
    ```

4. Validate the installation
    ```bash
    tanzu version
    tanzu plugin list
    ```