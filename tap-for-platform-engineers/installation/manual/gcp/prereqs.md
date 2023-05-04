# Manually Install TAP on GCP - Prerequisites <!-- omit from toc -->

- [GCP Preparations](#gcp-preparations)
- [Jump Host Setup](#jump-host-setup)
  - [Connect To Kubernetes](#connect-to-kubernetes)
  - [Service Account Key](#service-account-key)
- [Download Software](#download-software)

---

## GCP Preparations

Create the following GCP resources manually or use this [Terraform](https://github.com/unofficial-guide-to-tap/terraform/tree/main/gcp) project to automate the setup.

1. Create a **Virtual Private Cloud** (VPC)
2. Create a **Google Kubernetes Engine** (GKE) cluster in the VPC (check official docs for cluster requirements)
3. Create a **Cloud DNS** Managed Zone (or external DNS service)
4. Have access to **Google Container Registry** (GCR) (or external container registry)
5. A **Service Account Key** to be used by TAP 
6. An Ubuntu based **Google Compute Engine Instance** in the VPC that will be used as a jump host

<!--
END: ## Prepare The Infrastructure
-->

## Jump Host Setup

Before you proceed, install the following components:

| Software | Version | Check |
|---|---|---|
| Docker | >= 20.10 | `docker version` |
| Kubectl | >= 1.25  | `kubectl version --client` |
| gcloud | latest  | `gcloud version` |

### Connect To Kubernetes

```bash
gcloud auth login
gcloud container clusters get-credentials CLUSTER_NAME \
  --region REGION \
  --project PROJECT_ID
```

### Service Account Key

Create a copy of the Service Account key at `$HOME/key.json`.
```bash
vim $HOME/key.json
```

## Download Software
Download the following artifacts from [Tanzu Network](https://network.tanzu.vmware.com/) to your jump host.

| Artifact | Version  | Notes |
|---|---|---|
| Cluster Essentials for Linux| 1.5. | Contains [Carvel](https://carvel.dev/) tools |
| Cluster Essentials Bundle YAML | 1.5 | Contains the SHA hash |
| Tanzu Framework | 1.5.0 | This is the name of the Tanzu CLI which is the primary interface for platform engineers and application teams to interact with TAP. The software bundle is part of the "Tanzu Application Platform" product in Tanzu Network |


</br>

---
Next: [Installation](tap-for-platform-engineers/installation/manual/gcp/install.md)