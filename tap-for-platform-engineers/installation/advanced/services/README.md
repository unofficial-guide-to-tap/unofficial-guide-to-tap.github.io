# Provide Data Services

TAP provides the following pieces related to data services:

* **Services Binding:** The service binding specification allows to "bind" services to applications. [Read more](https://servicebinding.io/) 
* **Crossplane:** Crossplane allows the integration of AWS, Azure or GCP (among other) as a declarative API [Read more](https://www.crossplane.io/)
* **Services Toolkit:** The services toolkit is TAP's glue between services operators and application teams that allows them to discover, claim or provision services. [Read more](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/services-toolkit-concepts-service-consumption.html)

Using the components mentioned above, we can create a self-service experience while maintaining the options to either provision services dynamically and on-demand or have full control over the process. 

This guide covers the following use cases:

* **[External Services](tap-for-platform-engineers/installation/advanced/services/external/README.md):** Integrate external services such as databases managed by your DB team into TAP for consumption.
* **[Static Provisioning](tap-for-platform-engineers/installation/advanced/services/static/README.md):** Manually deploy services on Kubernetes and expose them to application teams for consumption.
* **[Dynamic Provisioning](tap-for-platform-engineers/installation/advanced/services/dynamic/README.md):** Let application teams manage the lifecycle of services and trigger service deployments on-demand.
