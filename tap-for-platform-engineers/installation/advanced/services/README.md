# Services

1. [Overview](#overview)
2. [Services Toolkit](./services-toolkit.md)

---

## Overview 

As a platform engineer providing TAP to your customers, you will be sooner or later be asked to provide access to services such as database, message queues or caches. This guide, covers the steps necessary to add this functionality to your developer platform.

### External Services

Connecting an application to existing services outside of TAP, works in the same way as it did before. Applications can use their cluster's connectivity to connect to the outside world in order to connect to e.g. a database. App teams may 

- parameterize their apps with environment variables
- hardcode some parameters into the code, or 
- use the [Service Binding](https://servicebinding.io/) specification to declaratively bind a service to their app

As a platform engineer, you are not involved in any of this. Check the [TAP For Application Teams](../../../../tap-for-app-teams/README.md) track of this guide, for detailed information.

### Service Lifecycle Management

You may decide to provide services as part of the TAP platform to allow a smoother "as a service" experience to developers. TAP does not provide e.g. Postgres or RabbitMQ by itself but integrated other components to do the heavy lifting of lifecycling service instances. 

Depending on the cloud platform you're running on, different options exist to add services capabilities to TAP. Follow one of the links below to learn how to install and use the respective integration.

| Method | Cloud Platform |
|---|---|
| [Tanzu SQL](https://docs.vmware.com/en/VMware-Tanzu-SQL) | any |
| [Tanzu RabbitMQ](https://docs.vmware.com/en/VMware-Tanzu-RabbitMQ-for-Kubernetes/) | any |
| [Crossplane](https://www.crossplane.io/) | any |
| [Config Connector](https://cloud.google.com/config-connector/docs/overview) | GCP |