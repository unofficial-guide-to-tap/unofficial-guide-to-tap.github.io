# SSL/TLS With Contour, Cert Manager And Letsencrypt & CloudDNS

This guide will setup TLS termination at the Kubernetes Ingress and make sure the services TAP GUI and any deployed `Workload`s are secured by default. The `Certificate` will be handled by Cert Manager and generated via Letsencrypt and a DNS-01 challenge via Google Cloud DNS.

> The `values.yaml` supports a `shared.ingress_issuer` attribute which causes CNRS and other services to use a specified `ClusterIssuer` to create `Certificate`s. CNRS will generate a unique host name for each deployed app and therefore will use the `ClusterIssuer` each time an application gets deployed. Letsencrypt secures their API (production, not staging) with a rate limit whic - once the limit is reached - will lock the account out for about a week! For this reason, in this guide we create a single `Certificate` that will work for all endpoints including applications (wildcard certificate) manually and reference it in the `values.yaml`.

## Setup Parameters

In this section, we set up some environment variables that will be referenced down the line in the installation steps.

```bash
GCP_PROJECT_ID="YOUR_PROJECT_ID"
LETSENCRYPT_EMAIL="YOUR_EMAIL_ADDRESS"
LETSENCRYPT_ORG_NAME="TAP Demo"
TAP_DOMAIN="tap.example.com"
```

You may e.g. copy the code above, edit and execute it in your shell using `EDITOR=vim fc`. Alternatively, save the copy above to a file like `ssl-params.sh` and load it into your shell with `source ssl-params.sh`. The latter makes it easier to load them again after you have exited your shell.



## Create Cluster Issuers

1. Create a `Secret` with a GCP service account key.

   This will allow Cert Manager to interact with Google Cloud DNS to perform the DNS-01 challenge.
    ```bash
    KEY_JSON_BASE64=$(cat $HOME/key.json | base64 -w0)
    ```
    ```bash
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
   
    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-staging-dns
    spec:
      acme:
        server: https://acme-staging-v02.api.letsencrypt.org/directory
        email: $LETSENCRYPT_EMAIL
        privateKeySecretRef: 
          name: letsencrypt-staging-dns-account-key
        solvers:
          - dns01:
              cloudDNS:
                serviceAccountSecretRef:
                  name: clouddns-dns01-solver-svc-acct
                  key: key.json
                project: $GCP_PROJECT_ID
            selector:
              dnsZones:
              - $TAP_DOMAIN
    EOF
    ```
    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-dns
    spec:
      acme:
        server: https://acme-v02.api.letsencrypt.org/directory
        email: $LETSENCRYPT_EMAIL
        privateKeySecretRef: 
          name: letsencrypt-dns-account-key
        solvers:
          - dns01:
              cloudDNS:
                serviceAccountSecretRef:
                  name: clouddns-dns01-solver-svc-acct
                  key: key.json
                project: $GCP_PROJECT_ID
            selector:
              dnsZones:
              - $TAP_DOMAIN
    EOF
    ```

## Generate Certificates

1. Create the `Certificate`

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: tap-cert
      namespace: tanzu-system-ingress
    spec:
      commonName: "*.$TAP_DOMAIN"
      dnsNames:
      - "*.$TAP_DOMAIN"
      issuerRef:
        kind: ClusterIssuer
        name: letsencrypt-staging-dns
      subject:
        organizations:
        - $LETSENCRYPT_ORG_NAME
      secretName: tap-cert
    EOF
    ```

2. Wait until `Certificate` was signed (READY=true)
    ```bash
    kubectl -n tanzu-system-ingress get certs
    ```

3. Switch to Letsencrypt production

    Knowing we're able to create certificates for our domain now, we can safely switch to Letsencrypt's production API to receive a proper certificate. This intermediate step prevents us from hitting Letsencrypts rate limit in which case we would be locked out of Letsencrypt for a couple of days.

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: tap-cert
      namespace: tanzu-system-ingress
    spec:
      commonName: "*.$TAP_DOMAIN"
      dnsNames:
      - "*.$TAP_DOMAIN"
      issuerRef:
        kind: ClusterIssuer
        name: letsencrypt-dns
      subject:
        organizations:
        - $LETSENCRYPT_ORG_NAME
      secretName: tap-cert
    EOF
    ```

    As before, wait for the `Certificate` to be successfully created.

4. Allow using this `Certificate` from other `Namespace`s 
    
   Check the [documentation](https://projectcontour.io/docs/1.24/config/tls-delegation/) for more information on `TLSCertificateDelegation`s.

    ```bash
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

    ```yaml
    cnrs:
      ingress_issuer: ""
      default_tls_secret: "tanzu-system-ingress/tap-cert"
      domain_template: "{{.Name}}-{{.Namespace}}.{{.Domain}}"

    tap_gui:
      tls:
        secretName: tap-cert
        namespace: tanzu-system-ingress
    ```

2. Update TAP

    ```bash
    tanzu package installed update tap \
      -n "tap-install" \
      -p tap.tanzu.vmware.com \
      -v "1.5.0" \
      --values-file values.yaml \
      --wait="false"
    ```

## Validate

### Check TAP GUI Certificate

```bash
openssl s_client -showcerts \
  -servername tap-gui.$TAP_DOMAIN \
  -connect tap-gui.$TAP_DOMAIN:443 < /dev/null
```

### Check Workload Certificate

If you alread created a test workload, make sure to delete it first. It needs to be redeployed for the certificate to become effective.


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

3. Check certificate

    ```bash
    openssl s_client -showcerts \
      -servername petclinic-test.$TAP_DOMAIN \
      -connect petclinic-test.$TAP_DOMAIN:443 < /dev/null
    ```