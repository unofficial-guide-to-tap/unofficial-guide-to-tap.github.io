# SSL/TLS With Contour And External Certificates

Some organizations follow very strict rules when it comes to certificates and put the generation and management of those into a single team's hands only. In that case, you will need to follow your organization's processes to get your hands on a Certificate.

## Certificate Signing Request

We're not going to document the way to create a CRS as this is likely to be different in your organization anyway. Make sure your certificate get issued for the following domains and replace `DOMAIN` with your base domain for your TAP installation. Something like `tap.example.com`.

* `*.DOMAIN`
* `*.cnrs.DOMAIN`

For the rest of this guide, we will assume you have your certificate in a file called `tls.crt` and the private key stored in `tls.key`.


## Install The Certificate

1. Create `Secret` with the certificate and private key

    ```bash
    kubectl -n tanzu-system-ingress create secret tls tap-cert --cert=tls.crt --key=tls.key
    ```

2. Allow using this `Secret` from other `Namespace`s 
    
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

2. Add the CA cert to your `values.yaml`

    ```yaml
    shared:
      ca_cert_data: |
        -----BEGIN CERTIFICATE-----
        MIIFKTCCBBGgAwIBAgISA7mVLxZ3d4GQTV/lE4CDC4gtMA0GCSqGSIb3DQEBCwUA
        ...
    ```

3. Update TAP

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
DOMAIN="..."
```
```bash
openssl s_client -showcerts \
  -servername tap-gui.$DOMAIN \
  -connect tap-gui.$DOMAIN:443 < /dev/null
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
    DOMAIN="..."
    ```
    ```bash
    openssl s_client -showcerts \
      -servername petclinic-test.$DOMAIN \
      -connect petclinic-test.$DOMAIN:443 < /dev/null
    ```