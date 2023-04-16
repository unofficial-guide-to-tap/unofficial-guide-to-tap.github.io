# Before You Start

To install and effectively use Tanzu Application Platform, it is essential to have a basic understanding of Kubernetes. Kubernetes is an open-source container orchestration platform that automates deploying, scaling, and managing containerized applications. It is widely used in the industry and forms the backbone of many cloud-native applications.

- [Before You Start](#before-you-start)
  - [Kubernetes Basics](#kubernetes-basics)
  - [Carvel Basics](#carvel-basics)
    - [ytt](#ytt)
    - [kapp-controller](#kapp-controller)
  - [Tanzu Packages](#tanzu-packages)

## Kubernetes Basics

Here are some recommended learning resources to help you grasp the fundamentals of Kubernetes and become familiar with Tanzu Application Platform and related technologies:

* [KubeAcademy](https://kube.academy/): KubeAcademy is a free, product-agnostic Kubernetes and cloud-native technology education platform. It offers a range of courses for beginners, intermediates, and advanced users. Start with the "Kubernetes 101" course to learn the basics.

* [Tanzu Academy](https://tanzu.vmware.com/academy): Tanzu Academy is a learning platform from VMware that focuses on VMware Tanzu products and services. It offers various courses, videos, and labs to help you understand and effectively use Tanzu Application Platform.

* [Spring Academy](https://spring.io/training): Spring Academy is the official learning platform for Spring Framework, a popular Java-based framework for building enterprise applications. As Tanzu Application Platform integrates well with Spring Boot and Spring Cloud, it is a great idea to explore the Spring ecosystem to build, deploy, and manage your applications efficiently.

* [Kubernetes Documentation](https://kubernetes.io/docs/home/): The official Kubernetes documentation is an excellent resource to learn about Kubernetes concepts and components. It also provides step-by-step guides and tutorials for various tasks.

* [Kubernetes: Up and Running](https://www.amazon.com/Kubernetes-Running-Dive-Future-Infrastructure/dp/1492046531): This book, written by Kelsey Hightower, Brendan Burns, and Joe Beda, provides a comprehensive introduction to Kubernetes and its ecosystem. It covers core concepts, patterns, and best practices for deploying and managing containerized applications.

By leveraging these resources and investing time in learning, you'll be well-equipped to install and use Tanzu Application Platform effectively. Remember to practice the concepts you learn to reinforce your understanding and gain hands-on experience.

## Carvel Basics

[Carvel](https://carvel.dev/) is a suite of open-source tools designed to simplify and streamline the management of Kubernetes resources and configurations. It is an essential component for installing and managing the Tanzu Application Platform, as it helps users create, deploy, and maintain Kubernetes applications in a consistent and reliable manner. The Carvel tools address various challenges associated with working with Kubernetes resources, including versioning, packaging, templating, and more.

The Carvel suite includes the following tools:

* [ytt](https://carvel.dev/ytt/): A YAML templating tool that enables users to create reusable, parameterized configurations for Kubernetes resources. ytt leverages Python-like syntax embedded within YAML to introduce dynamic capabilities.

* [kbld](https://carvel.dev/kbld/): A tool that simplifies the management of container images in Kubernetes configurations. kbld assists in building, pushing, and locating container images, thus ensuring that the correct image references are used during deployment.

* [kapp](https://carvel.dev/kapp/): A deployment tool that manages the lifecycle of Kubernetes applications. kapp handles the installation, updates, and deletion of resources while providing detailed insights into the changes made during each step.

* [kapp-controller](https://carvel.dev/kapp-controller/): A Kubernetes controller that automates application lifecycle management. It uses custom resources such as **App** and **PackageRepository** to manage application deployments, versioning, and updates. kapp-controller leverages the capabilities of the other Carvel tools, like ytt for templating and kapp for applying changes, to provide a seamless and declarative approach to deploying and managing applications on Kubernetes clusters.

* [imgpkg](https://carvel.dev/imgpkg/): A packaging tool that bundles and transfers Kubernetes resources and associated images. imgpkg creates immutable and shareable bundles, which simplifies the distribution and versioning of Kubernetes applications.

* [vendir](https://carvel.dev/vendir/): A tool for managing vendored repositories and directories by enabling users to declaratively define the contents they need from external sources. It ensures reproducible and versioned access to external resources, such as Git repositories or Helm charts.

The Carvel tools are important for installing and managing the Tanzu Application Platform because they provide a cohesive and streamlined approach to working with Kubernetes resources. They ensure that configurations are maintainable, scalable, and easy to understand. Additionally, they promote best practices for deploying and managing Kubernetes applications, thus fostering a reliable and consistent experience across teams and environments.

By using the Carvel tools, you can simplify the process of deploying and maintaining Tanzu Application Platform components, making it easier to adopt and manage cloud-native applications on Kubernetes.

### ytt

To experiment with ytt and learn more about its capabilities, you can use the ytt [Playground](https://playground.tanzu.vmware.com/ytt). The playground provides an interactive environment where you can write, test, and share ytt templates without having to install any software on your local machine. This is an excellent resource to get started with ytt and familiarize yourself with its features and syntax.

### kapp-controller

## Tanzu Packages
