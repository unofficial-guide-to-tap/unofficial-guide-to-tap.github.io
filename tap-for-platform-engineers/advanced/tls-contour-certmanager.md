# SSL/TLS With Contour And Cert Manager

## Configure Cert Manager

```
---
apiVersion: v1
kind: Secret
metadata:
  name: clouddns-dns01-solver-svc-acct
  namespace: cert-manager
data:
  key.json: KEY_JSON_BASE64
```

```
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging-dns
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: YOUR_EMAIL_ADDRESS
    privateKeySecretRef: 
      name: letsencrypt-staging-dns-account-key
    solvers:
      - dns01:
          cloudDNS:
            serviceAccountSecretRef:
              name: clouddns-dns01-solver-svc-acct
              key: key.json
            project: PROJECT_ID
        selector:
          dnsZones:
          - DOMAIN

---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: YOUR_EMAIL_ADDRESS
    privateKeySecretRef: 
      name: letsencrypt-staging-dns-account-key
    solvers:
      - dns01:
          cloudDNS:
            serviceAccountSecretRef:
              name: clouddns-dns01-solver-svc-acct
              key: key.json
            project: PROJECT_ID
        selector:
          dnsZones:
          - DOMAIN
```

## Generate Certificates

```
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tap-cert
  namespace: tanzu-system-ingress
spec:
  commonName: *.DOMAIN
  dnsNames:
  - *.DOMAIN
  issuerRef:
    kind: ClusterIssuer
    name: letsencrypt-staging-dns
  subject:
    organizations:
    - ORGANIZATION_NAME
  secretName: tap-cert

---
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
```


## Configure TAP for TLS

```
cnrs:
  ingress_issuer: ""
  default_tls_secret: "tanzu-system-ingress/tap-cert"

learningcenter:
  ingressSecret:
    secretName: learningcenter-cert

tap_gui:
  tls:
    secretName: tap-cert
    namespace: tanzu-system-ingress

accelerator:
  ingress:
    enable_tls: true
  tls:
    secret_name: tap-cert
    namespace: tanzu-system-ingress

appliveview:
  ingressEnabled: true
  tls:
    secretName: tap-cert
    namespace: tanzu-system-ingress

appliveview_connector:
  backend:
    sslDeactivated: false

metadata_store:
  ingress_enabled: true
  tls:
    secretName: tap-cert
    namespace: tanzu-system-ingress
```