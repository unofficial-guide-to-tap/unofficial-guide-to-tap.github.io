As a **Platform Engineer**, I want to
# Add Data Services

1. [Kubernetes Extensions](#kubernetes-extensions)
    1. [Tanzu SQL for Kubernetes](./tanzu-sql.md)
    2. [Tanzu RabbitMQ for Kubernetes](./tanzu-rmq.md)
    3. [Crossplane](https://www.crossplane.io/)
    4. [Config Connector](https://cloud.google.com/config-connector/docs/overview)
2. [Services Toolkit](./services-toolkit.md)
---

As a platform engineer providing TAP to your customers, you will be sooner or later be asked to provide access to services such as database, message queues or caches. This guide, covers the steps necessary to add this functionality to your developer platform.

## Kubernetes Extensions

The most common way to integrate data services in TAP is to lifecycle those services directly on Kubernetes. This is normally done by extending the Kubernetes API by resource types specific to the services to manage. Very much like `Pod` or `Deployment` you now have resources like `Postgres` or `RabbitmqCluster` available in your Kubernetes cluster. 

Depending on the cloud platform you're running on, different options exist to add data services capabilities to TAP. The following list is not meant to be complete. It only represents those, we have tested with. 

| Extension | Cloud Platform | Included in TAP |
|:---|---|---|
| [Tanzu SQL for Kubernetes](./tanzu-sql.md) | any | NO |
| [Tanzu RabbitMQ for Kubernetes](./tanzu-rmq.md) | any | NO |
| [Crossplane](https://www.crossplane.io/) | any | YES |
| [Bitnami](https://bitnami.com/) | any | YES |
| [Config Connector](https://cloud.google.com/config-connector/docs/overview) | GCP | NO |

Follow one of the links above to learn how to install and use the respective integration.
