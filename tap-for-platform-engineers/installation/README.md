Installation
============

1. [Manual Installation](./manual/README.md)
    1. [Google Cloud Platform](./manual/gcp/README.md)
    2. Microsoft Azure
    3. Amazon Web Services
2. GitOps Installation
3. [Advanced Configurations](./advanced/README.md)
---

While the details of the installation process vary depending on the platform in use (e.g. AWS, GCP, Azure), the general procedure remains the same:

1. Set up a Kubernetes cluster with access to a container registry
3. Mirror installation packages from Tanzu Network to your registry
4. Install Kapp Controller on the Kubernetes cluster
5. Register the package mirror as `PackageRepository`
6. Craft the `values.yaml` to parameterize the `Package` called "tap"
7. Install the "tap" `Package`

**Minimal Installation**

In this guide, we strive to give a to-the-point installation guide that requires the reader to run a minimum number of steps to get TAP up and running. All advanced installation topics will be covered in subsequent chapters that build on exactly this minimal installation.

**Jump Host**

All installation steps are designed to be run from an **Ubuntu** jump host where applicable for the following reasons:

1. **Consistency**: We want to provide the exact command to be used for a certain step. Maintaining that across all possible operating systems a reader my be installing from, is not manageable for us at this point. 

2. **Speed**: While not strictly part of a minimal installation, we include steps to mirror the package repositories which creates a significant amount of traffic. Running this from a jump host in the datacenter or public cloud will tremendously speed up the installation experience.
