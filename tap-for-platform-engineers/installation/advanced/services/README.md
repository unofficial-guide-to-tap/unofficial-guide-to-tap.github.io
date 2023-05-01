# Data Services In TAP

1. [Overview](#overview)
2. [Kubernetes Extensions](#kubernetes-extensions)
    1. [Tanzu SQL for Kubernetes](./tanzu-sql.md)
    2. [Tanzu RabbitMQ for Kubernetes](./tanzu-rmq.md)
    3. [Crossplane](https://www.crossplane.io/)
    4. [Config Connector](https://cloud.google.com/config-connector/docs/overview)
3. [Services Toolkit](./services-toolkit.md)
---

As a platform engineer providing TAP to your customers, you will be sooner or later be asked to provide access to services such as database, message queues or caches. This guide, covers the steps necessary to add this functionality to your developer platform.

## Overview

Ultimately, the goal is to allow applications to connect to services. A `Workload` resource can be configured to attach those services via `ServiceClaim`s which are part of the [Service Bindings](https://servicebinding.io/) specification. Therefore, for a platform engineer running TAP the goal has to be to provide that service binding compatible resource so it can be attached to the application. 

For `ServiceBinding`s to work, the following requirements need to be fulfilled:

1. A Kubernetes resource that has a `status.binding.name` property pointing to a compatible `Secret` or only the compatible `Secret`

    This depends on the Kubernetes operator or other sort of extension that is being used to deploy a specific service. While Tanzu SQL and Tanzu RabbitMQ, for instance, are compatible other such as the MongoDB operator are not (yet). This does not mean those extensions cannot be used, but there are additional steps necessary to integrate into the Service Binding mechanism.

2. That Kubernetes resource has to be in the same `Namespace` as the `ServiceBinding` and the application to bind to.

    For TAP, this means all service instance resources have to be deployed into the developer's namespace by either giving them full control over the lifecycle management of the respective service or by significantly adding to the complexity of managing a fleet of services by a centralized services operators team. That team would need access to all developer `Namepspace`s and their service instances would be scattered across `Namespace`s and potentially clusters.

TAP includes the `Services Toolkit` which adds support for the following:

- **Segregation of concerns**

  Services and applications can be managed by different teams without scattering services across application namespaces. A single team of services operators can easily manage a large fleet of services.

- **Self-service for application teams**

  Application teams may discover available services and service classes and use claims to consume an existing or trigger the dynamic provisioning of a service.

- **Abstract the underlying mechanism to provision services**

  While there are many ways to manage services with or without Kubernetes, application teams aren't confronted with those details. Service operator teams may use whatever mechanism they deem useful without revealing those details to app teams.


## Kubernetes Extensions

The most common way to integrate data services in TAP is to lifecycle those services directly on Kubernetes. This is normally done by extending the Kubernetes API by resource types specific to the services to manage. Very much like `Pod` or `Deployment` you now have resources like `Postgres` or `RabbitmqCluster` available in your Kubernetes cluster. 

Depending on the cloud platform you're running on, different options exist to add data services capabilities to TAP. The following list is not meant to be complete. It only represents those, we have tested with. 

| Extension | Cloud Platform |
|---|---|
| [Tanzu SQL for Kubernetes](./tanzu-sql.md) | any |
| [Tanzu RabbitMQ for Kubernetes](./tanzu-rmq.md) | any |
| [Crossplane](https://www.crossplane.io/) | any |
| [Config Connector](https://cloud.google.com/config-connector/docs/overview) | GCP |

Follow one of the links above to learn how to install and use the respective integration.