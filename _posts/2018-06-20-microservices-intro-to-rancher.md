---
layout:     post
title:      "What a Microservice wants, What a Microservice needs; Understanding Rancher"
date:       2018-04-10 01:02:58 +0530
comments:   true
---

With increasing adoption of [Docker](https://trends.google.com/trends/explore?date=all&q=docker) and its orchestrator platforms the microservices are good way to ship software where-ever applicable. Learning some bits about them which experimenting with Rancher.

### Containers have their benefits
It is easy to understand why Docker adoption is high. It gives:
- Isolation & consistency
- Docker repository
- Shipping packages with all their dependencies
- Take a base image and build out a way from there
- At the end, it is a just software we are shipping, it is computer resources we are using, it is services we are deploying(read: hosting). The app needs to customised and business specific; not deployment.

### Microservices have their benefits (if we can ignore the cons)
- Scaling is replicating monolith if not microservice; so as far as this post is concerned, this is enough of a benefit to adapt microservices wherever we can; now that we have containers to ship them.

### What a microservice wants
Any service wants itself to be served, so is a microservice. It wants to be called/accessible. That asks it to be easy to integrate.

Medium of communication is an overhead sometimes compared to monolith - so, it wants to be fast for low latency calls. Some http paradigm like gRPC tries to make it better than REST.

A microservice wants to have its own lifecycle(read: life of its own). It means it has freedom of programing language, freedom of deployment cycle. It wants to be easy to ship. (Thanks Docker!)

### What a microservice needs
It needs to one thing but do well.

A service needs to be hosted. A stack where it can live! (apart from that, it also needs everything that is there for it to stay microservice; [they tend to be like monolith very fast](https://www.youtube.com/watch?v=X0tjziAQfNQ))
Hosting comes with a price: it needs to have health checks(everybody needs a doctor), networking(to socialize), monitoring(to diagnose and early alerts), scaling(everybody needs to adapt), updating/deployments(everybody needs to grow)
This note is about understanding [Rancher](https://rancher.com/); a container orchestrator platform to host and manage microservices deployments.

### Understanding Rancher
Rancher helps manage containers, helps manage microservices; helps manage microservices which are running on containers. Makes it easy to understand cluster with its micro-services and coordinates container distribution. Rancher makes things happen with containers.

#### Anatomy of Rancher
In a nutshell, Rancher is abstraction over resource pool to distribute(and schedule) containers. Distribute them to run host agnostic but group them to form a microservice.
Rancher provides UI to manage the resources and deployment as well along with simple APIs to interact with rancher cluster.

##### Host
The unit of resource is host. A Linux machine (it can be virtual of physical machine).
It provides: CPU, Memory, Network, Storage.
Rancher can(read:will) run containers on the host to use these resources.

##### Stack
Stack is where services are deployed. Serves two purposes:
- Logical grouping of Hosts. One can configure the containers to be hosted on group of Hosts. (Affinity)
- Stack of services. Running collection of microservices on a stack.

##### Service
Service is the microservice. It will have its own load balancer rule and a url to be accessed from.
Service is collection of containers. All containers in a service runs a same Docker image. The multiple containers are for the scale of the service. The scale can be desired as will and needs. One service can have scale of 3 containers and other service can have scale of 1 container.
Scaling: If we change the scale from 3 to 4 - Rancher will schedule a new container to any host from the stack and run the container image. This makes it super easy to horizontally scale the service. Service load balancer and HA proxy module of Rancher allows to easily upgrade services without downtime.

##### Container
Container is unit of work. Rancher manages container lifecycle. So, container has state; either of: Started, Initialising, Running, Killed, Stopped.
Containers can optionally have access of external storage through NFS drivers.
Once it is Running it will receive traffic from service load balancer.
Rancher provides support for container health-check. And it also manages the desired state of the service. Rancher takes care of managing networking for containers and between container and service.

Typical cluster topology running containers on rancher stack: 

![https://i.imgur.com/eL0PN2h.jpg](https://i.imgur.com/eL0PN2h.jpg)

### References:
1. [https://docs.docker.com/](https://docs.docker.com/)
1. [https://rancher.com/docs/rancher/v1.6/en/rancher-services/](https://rancher.com/docs/rancher/v1.6/en/rancher-services/)
1. [https://rancher.com/docs/](https://rancher.com/docs/)
1. [https://www.slideshare.net/ConnerSwann/an-introduction-to-rancher](https://www.slideshare.net/ConnerSwann/an-introduction-to-rancher)
1. [https://www.youtube.com/watch?v=X0tjziAQfNQ](https://www.youtube.com/watch?v=X0tjziAQfNQ)
1. [https://medium.com/ibm-watson-data-lab/painless-container-management-with-rancher-2-0-kubernetes-and-ibm-cloud-5a14ac2d4ccc](https://medium.com/ibm-watson-data-lab/painless-container-management-with-rancher-2-0-kubernetes-and-ibm-cloud-5a14ac2d4ccc)
1. [https://www.martinfowler.com/articles/microservices.html](https://www.martinfowler.com/articles/microservices.html)
1. [https://dzone.com/articles/top-trends-machine-learning-microservices-containe](https://dzone.com/articles/top-trends-machine-learning-microservices-containe)
