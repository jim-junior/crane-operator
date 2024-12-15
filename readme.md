# Building a Kubernetes Operator | A Practical Guide

![](https://www.epsglobal.com/Media-Library/EPSGlobal/Blog/kubernets2.jpg)

One of those great revolutionary technologies that have transformed how developers think of and Interact with cloud infrastructure is Kubernetes. Initially Developed at Google, Kubernetes, also known as K8s, is an open source system for automating deployment, scaling, and management of containerized applications. It is Designed on the same principles that allow Google to run billions of containers a week, Kubernetes can scale without increasing your operations team.

But however Amazing and Powerful kubernetes might be, It remains just a container Orchestration technology at the core and will never solve all problems that Software engineers face in the complex world of Software development and Deployment or DevOps. To Solve this Kubernetes provides ways it can be Extended and customized to meet your team's needs. It Provides Client Libraries in many languages. But even better are Kubernetes Operators.

A formal definition of a kubernetes operator can be:

> A Kubernetes Operator is a method of automating the management of complex applications on Kubernetes. It extends Kubernetes' capabilities by using custom resources and controllers to manage the lifecycle of an application, automating tasks such as deployment, scaling, and updates. Operators encode the operational knowledge of an application into Kubernetes, making it easier to manage stateful or complex workloads with less manual intervention.

In this article. We shall go through a guide to get started building your own Custom kubernetes Operator. We shall cover different topics like Custom Resource Definitions, Controllers and look at the Kubernetes Controller Runtime.

## Prerequisites

There are a few thing we need to know and have before continuing with this tutorial

- A good understanding of Kubernetes and how to use it.
- Programming Knowledge in GoLang
- Access to a Kubernetes Cluster (You can try it locally using Minikube or Kind)

## Setting up your Environment

To set up your environment you will first need to have Go installed. The Kubernetes Golang Client usually requires a specific Go version so depending on your time of reading this article it might have changed but for now i will use go1.21.6. To know what version you are using, run

```bash
go version
```

Next you will need to have access to a Kubernetes cluster but for development purposes I would advise you use a local cluster from tools like Minikube or Kind. You can visit their websites for the installation steps.

We shall then have to initialize our module that we will use for development. Run

```bash
go mod init https://github.com/jim-junior/crane-operator
```

Before we dive right into code. There are certain key concepts you will need to know so let's look at those first.

## Concepts

### Custom Resource Definitions (CRDS)

To understand CRDs you need to first know what a resource is. Pods, Deployments, Services etc are all resources. Formally

> A resource is an endpoint in the [Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/) that stores a collection of [API objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/#kubernetes-objects) of a certain kind; for example, the built-in pods resource contains a collection of Pod objects.

Resources are in built within the Kubernetes API. But in our case, we said one of the main resons we build operators is to handle custom problems that Kubernetes does not solve out of the box. In some cases we might need to define our own resource objects. Forexample. Imagine we are building an operator that manages postgreSql databases, we would like to provide an API to define configurations of each database we initiate, we can do this by defining a Custom resource definition `PGDatabase` as an Object that stores the configuration of the database.

__Example of a Custom Resource Definition for demostration__

```yml
apiVersion: example.com/v1 # Every resource must have an API Version 
kind: PGDatabase # CRD Name
metadata:
  name: mydb
spec:
  config:
    port: 5432
    user: root
    dbname: mydb

  volumes:
  - pgvolume: /pgdata/

  envFrom: wordpress-secrets
```

Note: CRDs are just definitions of the actually objects or they just represent an actual object in the kubernetes cluster but are not the actual object. For example when you write a yaml file defining how a pod should be scheduled, that yaml just defines but its not the actual pod.

Therefore we can define a CRD as an extension of the Kubernetes API that is not necessarily available in a default Kubernetes installation. It represents a customization of a particular Kubernetes installation. Actually, many core Kubernetes functions are now built using custom resources, making Kubernetes more modular.

### Controllers

Next Lets look at Kubernetes Controllers. As we have seen above, CRDs represent objects or state of objects in the Kubernetes Cluster, but we need something to reconcile or transform that state into actual resources on the cluster, this is where Controllers come in.

> A Kubernetes controller is a software component within Kubernetes that continuously monitors the state of cluster resources and takes action to reconcile the actual state of the cluster with the desired state specified in resource configurations (YAML files).


![](https://iximiuz.com/writing-kubernetes-controllers-operators/kdpv.png)

Controllers are responsible for managing the lifecycle of Kubernetes objects, such as Pods, Deployments, and Services. Each controller typically handles one or more resource types and performs tasks like creating, updating, or deleting resources based on a declared specification.
Here's a breakdown of how a Kubernetes controller operates:

1. __Observe__: The controller watches the API server for changes to resources it's responsible for. It monitors these resources by querying the Kubernetes API periodically or via an event stream.
2. __Analyze__: It compares the actual state (current conditions of the resources) to the desired state (specified in configurations).
3. __Act__: If there’s a discrepancy between the actual and desired state, the controller performs operations to align the actual state with the desired state. For example, a Deployment controller might create or terminate Pods to match the specified replica count.
4. __Loop__: The controller operates in a loop, continuously monitoring and responding to changes to ensure the system’s resources are always in the desired state.

### Controller Runtime

Kubernetes provides a set of tools to build native Controllers and these are Known as the Controller Runtime.

Lets take a deep look into the Controller runtime because its what we shall be using to build out Operators

## A Deep look into the Controller Runtime

Controller Runtime is a set of libraries and tools in the Kubernetes ecosystem, designed to simplify the process of building and managing Kubernetes controllers and operators. It’s part of the Kubernetes Operator SDK and is widely used to develop custom controllers that can manage Kubernetes resources, including custom resources (CRDs).

Controller Runtime provides a structured framework for controller logic, handling many of the lower-level details of working with the Kubernetes API, so developers can focus more on defining the behavior of the controller and less on boilerplate code. It is written in Go and builds on the Kubernetes `client-go` libraries.

### Key Components of Controller Runtime

The Controller Runtime has several key components that streamline the process of building and running Kubernetes controllers. Together, these components create a robust framework for building Kubernetes controllers.

The Manager initiates and manages other components; the Controller defines reconciliation logic; the Client simplifies API interactions; the Cache optimizes resource access; Event Sources and Watches enable event-driven behavior; and the Reconcile Loop ensures continuous alignment with the desired state. These components make it easier to build controllers and operators that efficiently manage Kubernetes resources, allowing for custom automation and orchestration at scale.

#### Manager

The Manager is the main entry point of a controller or operator, responsible for initializing and managing the lifecycle of other components, such as controllers, caches, and clients. Developers usually start by creating a Manager in the main function, which acts as the foundational setup for the entire operator or controller.

It provides shared dependencies (e.g., the Kubernetes client and cache) that can be reused across controllers, allowing multiple controllers to be bundled within a single process.

The Manager also coordinates starting and stopping all controllers it manages, ensuring that they shut down gracefully if the Manager itself is stopped.

#### 1. Controller

The Controller is the core component that defines the reconciliation logic responsible for adjusting the state of Kubernetes resources. It is a control loop that monitors a cluster's state and makes changes to move it closer to the desired state

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*zacjsy7nznxEgFhxJFfC0w.png)

Each controller watches specific resources, whether built-in (e.g., Pods, Deployments) or custom resources (CRDs). It includes a `Reconcile` function that’s triggered whenever a resource changes, allowing the controller to bring the current state in line with the desired state.

Developers specify which resources the controller should watch, and Controller Runtime will automatically track and respond to events (like create, update, delete) for those resources. In a controller for managing custom `Foo` resources, the `Reconcile` function might create or delete associated resources based on Foo specifications.

#### 2. Client

The Client is an abstraction that simplifies interactions with the Kubernetes API, enabling CRUD operations on resources.

This component allows for easy creation, reading, updating, and deletion of Kubernetes resources.
The Client is integrated with the cache to provide efficient access to resources without overloading the API server.

Controller Runtime’s Client extends the basic Kubernetes `client-go` library, making API calls more straightforward by handling details like retries and caching behind the scenes.

Using the Client, developers can create a Pod directly from the controller logic with a single line of code, `client.Create(ctx, pod)`, without having to worry about raw API requests.

#### 3. Event Sources and Watches

Event Sources and Watches determine the resources that a controller will monitor for changes, providing a way to respond to specific events in the cluster.

Event Sources define what triggers the controller’s reconciliation loop, which can be based on changes in specific Kubernetes resources.
Watches monitor these resources, allowing the controller to act on create, update, or delete events as needed.

Developers can define multiple Watches for a single controller, which is useful if the controller’s behavior depends on multiple resources.

A controller managing a custom `App` resource might watch Pods, Services, and ConfigMaps, reacting to changes in any of these resources by adjusting the `App` accordingly.

#### 4. Reconcile Loop

The Reconcile loop is the heart of the controller, implementing the main logic that determines the steps to bring resources into the desired state.
Each controller’s Reconcile function checks the actual state of a resource and then applies necessary changes to make it conform to the desired state.
This loop continues indefinitely, with each reconciliation acting as a self-healing mechanism to correct any divergence from the specification.
The Reconcile loop is usually idempotent, meaning it can be repeated without causing unintended side effects, ensuring consistency even with frequent updates.
In a Reconcile function, the controller might find that a Deployment lacks the specified number of replicas, so it updates the Deployment configuration to match the desired replica count.

To prevent this article from becoming so long we shall cut it here and in part 2 we shall dive into some code and build a simple Kubernetes Operator that

## References

Brandon Philips, (November 3, 2016). Introducing Operators: Putting Operational Knowledge into Software. Internet Archive Wayback Machine. [https://web.archive.org/web/20170129131616/https://coreos.com/blog/introducing-operators.html](https://web.archive.org/web/20170129131616/https://coreos.com/blog/introducing-operators.html)

CNCF TAG App-Delivery Operator Working Group, CNCF Operator White Paper - Final Version. Github. [https://github.com/cncf/tag-app-delivery/blob/163962c4b1cd70d085107fc579e3e04c2e14d59c/operator-wg/whitepaper/Operator-WhitePaper_v1-0.md](https://github.com/cncf/tag-app-delivery/blob/163962c4b1cd70d085107fc579e3e04c2e14d59c/operator-wg/whitepaper/Operator-WhitePaper_v1-0.md)

Kubernetes Documentation, Custom Resources. [https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

Kubernetes Documentation, Controllers. [https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/](https://kubernetes.io/docs/concepts/architecture/controller/)
