# TAP on GCP - Prerequisites

<!-- TOC depthfrom:2 depthto:2 orderedlist:false -->

- [GCP Preparations](#gcp-preparations)
- [Jump Host Setup](#jump-host-setup)
- [Download Software](#download-software)

<!-- /TOC -->

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

```
gcloud auth login
gcloud container clusters get-credentials CLUSTER_NAME \
  --region REGION \
  --project PROJECT_ID
```

### Service Account Key

Create a copy of the Service Account key at `$HOME/key.json`.
```
vim $HOME/key.json
```

## Download Software
Download the following artifacts from [Tanzu Network](network.tanzu.vmware.com/) to your jump host.

| Artifact | Version  | Notes |
|---|---|---|
| Cluster Essentials for Linux| 1.4.1 | Contains [Carvel](https://carvel.dev/) tools |
| Cluster Essentials Bundle YAML | 1.4.1 | Contains the SHA hash |
| Tanzu Framework | 1.4.4 | This is the name of the Tanzu CLI which is the primary interface for platform engineers and application teams to interact with TAP. |

---
Next: [Installation](./install.md)