# Installation

## Overview
While the details of the installation process vary depending on the platform in use (e.g. AWS, GCP, Azure), the general procedure remains the same:

1. Set up a Kubernetes cluster with access to a container registry
2. Mirror installation packages from Tanzu Network to your registry
3. Install Kapp Controller on the Kubernetes cluster
4. Register the package mirror as `PackageRepository`
5. Craft the `values.yaml` to parameterize the `Package` called "tap"
6. Install the "tap" `Package`


## Principles

The installation guides follow a few principles that we believe are important to foster an understanding of what's happening and how things work.

-  **Minimal:** All our installation guides perform a **minimum installation** of TAP. We don't leave out any important steps but keep the configuration made to a minimum. Our **advanced topics** build on top of this minimum installation and help you extend TAP with more functionality and features.

- **Jump Host:** All installation steps are designed to be run from an **Ubuntu** jump host where applicable. This way we gain consistency througout all guides while improving speed when it comes to shifting around large amounts of data.

- **Single Document:** All steps are listed in a single document so that readers may follow along top to bottom without having break their flow my having to follow links and then return to the document. This comes at the expense of repeated documentation of steps and higher maintenance effort of the guide, but we think that's worth it.

- **Validation:** After every installation guide, we provide a validation procedure. This way readers can validate their environment is working as expected.