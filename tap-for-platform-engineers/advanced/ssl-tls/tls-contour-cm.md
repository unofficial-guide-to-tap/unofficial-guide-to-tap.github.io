# SSL/TLS With Contour And Cert Manager

This guide will setup TLS termination at the Kubernetes Ingress and make sure the services TAP GUI and any deployed `Workload`s are secured by default. The `Certificate` will be handled by Cert Manager and generated via Letsencrypt and a DNS-01 challenge via Google Cloud DNS.

## Configure Cert Manager

1. Create a `Secret` with a GCP service account key.

   This will allow Cert Manager to interact with Google Cloud DNS to perform the DNS-01 challenge.
    ```
    GOOGLE_APPLICATION_CREDENTIALS="$HOME/key.json"
    KEY_JSON_BASE64=$(cat $GOOGLE_APPLICATION_CREDENTIALS | base64 -w0)
    ```
    ```
    cat <<EOF | kubectl apply -f -
    apiVersion: v1
    kind: Secret
    metadata:
      name: clouddns-dns01-solver-svc-acct
      namespace: cert-manager
    data:
      key.json: $KEY_JSON_BASE64
    EOF
    ```

2. Create the `ClusterIssuer`

   The `ClusterIssuer` tells Cert Manager which `Certificate` requests to handle and how to handle them. We create one for the Letsencrypt Staging API and one for the Production API. This way we can play with it until it works before getting locked out.

    ```
    PROJECT_ID="..."
    EMAIL_ADDRESS="..."
    DOMAIN="..."
    ```
    ```
    cat <<EOF | kubectl apply -f -
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-staging-dns
    spec:
      acme:
        server: https://acme-staging-v02.api.letsencrypt.org/directory
        email: $EMAIL_ADDRESS
        privateKeySecretRef: 
          name: letsencrypt-staging-dns-account-key
        solvers:
          - dns01:
              cloudDNS:
                serviceAccountSecretRef:
                  name: clouddns-dns01-solver-svc-acct
                  key: key.json
                project: $PROJECT_ID
            selector:
              dnsZones:
              - $DOMAIN
    EOF
    ```
    ```
    cat <<EOF | kubectl apply -f -
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-dns
    spec:
      acme:
        server: https://acme-v02.api.letsencrypt.org/directory
        email: $EMAIL_ADDRESS
        privateKeySecretRef: 
          name: letsencrypt-dns-account-key
        solvers:
          - dns01:
              cloudDNS:
                serviceAccountSecretRef:
                  name: clouddns-dns01-solver-svc-acct
                  key: key.json
                project: $PROJECT_ID
            selector:
              dnsZones:
              - $DOMAIN
    EOF
    ```

## Generate Certificates

1. Create the `Certificate` resources

   By creating `Certificate`s Cert Manager triggers and attempts to generate a signed cert.

    ```
    DOMAIN="..."
    ORGANIZATION_NAME="..."
    ```

    ```
    cat <<EOF | kubectl apply -f -
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: tap-cert
      namespace: tanzu-system-ingress
    spec:
      commonName: "*.$DOMAIN"
      dnsNames:
      - "*.$DOMAIN"
      issuerRef:
        kind: ClusterIssuer
        name: letsencrypt-staging-dns
      subject:
        organizations:
        - $ORGANIZATION_NAME
      secretName: tap-cert
    EOF
    ```

2. Wait until `Certificate` was signed (READY=true)
    ```
    kubectl -n tanzu-system-ingress get certs
    ```

2. Allow using this `Certificate` from other `Namespace`s 
    
   Check the [documentation](https://projectcontour.io/docs/1.24/config/tls-delegation/) for more information on `TLSCertificateDelegation`s.

    ```
    cat <<EOF | kubectl apply -f -
    apiVersion: projectcontour.io/v1
    kind: TLSCertificateDelegation
    metadata:
      name: tap-cert-delegation
      namespace: tanzu-system-ingress
    spec:
      delegations:
        - secretName: tap-cert
          targetNamespaces:
            - "*"
    EOF
    ```



## Configure TAP for TLS

1. Apply the following changes to your `values.yaml`

    ```
    cnrs:
      ingress_issuer: ""
      default_tls_secret: "tanzu-system-ingress/tap-cert"

    tap_gui:
      tls:
        secretName: tap-cert
        namespace: tanzu-system-ingress
    ```

2. Update TAP

    ```
    tanzu package installed update tap \
      -n "tap-install" \
      -p tap.tanzu.vmware.com \
      -v "1.4.4" \
      --values-file values.yaml \
      --wait="false"
    ```

## Validate

### TAP GUI
```
DOMAIN="..."
```
```
openssl s_client -showcerts \
  -servername tap-gui.$DOMAIN \
  -connect tap-gui.$DOMAIN:443 < /dev/null
```

### Test Workload

If you alread created a test workload, make sure to delete it first. It needs to be redeployed for the certificate to become effective.


1. Create a developer namespace
    ```
    kubectl create ns --dry-run=client -o yaml test | kubectl apply -f -
    kubectl label namespaces test apps.tanzu.vmware.com/tap-ns=""
    ```

2. Deploy the workload
    ```
    tanzu apps workload create petclinic -n test \
      -l "app.kubernetes.io/part-of=petclinic" \
      -l "apps.tanzu.vmware.com/workload-type=web" \
      --build-env "BP_JVM_VERSION=17" \
      --git-repo https://github.com/spring-projects/spring-petclinic.git \
      --git-branch main
    ```

3. Check certificate

    ```
    DOMAIN="..."
    ```
    ```
    openssl s_client -showcerts \
      -servername petclinic-test.$DOMAIN \
      -connect petclinic-test.$DOMAIN:443 < /dev/null
    ```