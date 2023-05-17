1. Extract the downloaded archive

    ```bash
    mkdir cluster-essentials
    tar xvf tanzu-cluster-essentials-linux-amd64-1.5.0.tgz -C cluster-essentials
    cd cluster-essentials
    ```

2. Install the executables
    ```bash
    sudo install ./imgpkg /usr/local/bin/imgpkg
    sudo install ./kapp /usr/local/bin/kapp
    sudo install ./kbld /usr/local/bin/kbld
    sudo install ./ytt /usr/local/bin/ytt
    ```

3. Validate the installation
    ```bash
    imgpkg --version
    kapp --version
    kbld --version
    ytt --version
    ```