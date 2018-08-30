---
layout:     post
title:      "Microservice in Java (Part-2): Docker container for gRPC server"
date:       2018-08-18 12:23:58 +0530
comments:   true
---

Notes on Microservice implementation in Java. Creating Docker container for gRPC server. Following the recent trend to containerize the microservice server to make deployments easier using tools like Docker and Kubernetes.

### gRPC server
This is part-2 for the Java microservice. Recap a it of [first part](https://yogin16.github.io/2018/08/18/grpc-gradle-spring/):
- We have a hello world greeter service implemented in Java using gradle and spring
- The greeter service would listen to gRPC requests

### Main method
To execute Java main method via gradle we need to add a new task in our `build.gradle`:

```groovy
// execute main method for HelloMain
task execute(type:JavaExec) {
    main = "com.example.HelloMain"
    classpath = sourceSets.main.runtimeClasspath
}
```

### Docker
We want to create Docker image for above service. As a common practice for container images, it should:
- have a base image
- have all java the dependencies while building the image
- start the server on the docker run
- have a tag for versioning the release

We already used gradle for building proto and managing dependencies for Java server, we can use gradle in Docker image as well.

#### Dockerfile
Above gradle task simplifies the Docker image:
- using java:8 base image to get the Java JDK we would need
- installing gradle in the image
- `gradle build` would install all the java dependencies
- execute above gradle task of main method on docker container start

```Dockerfile
FROM java:8
RUN mkdir -p /grpc-hello
COPY ./ /grpc-hello
WORKDIR /grpc-hello
RUN apt-get -y update && \
    apt-get install -y --no-install-recommends wget unzip
RUN wget https://services.gradle.org/distributions/gradle-3.4.1-bin.zip
RUN mkdir /opt/gradle
RUN unzip -d /opt/gradle gradle-3.4.1-bin.zip
ENV PATH="/opt/gradle/gradle-3.4.1/bin:${PATH}"
RUN gradle build
CMD ["gradle", "execute"]
```

To build docker image from above Dockerfile:

```bash
docker build -f ./tools/Dockerfile
```

This docker image is published to docker hub: [yogin16/grpc-hello](https://hub.docker.com/r/yogin16/grpc-hello/)

So, to start the grpc server to any server running Docker we will just have to do:

```bash
docker pull yogin16/grpc-hello
docker run -p 10000:10000 yogin16/grpc-hello
```

Docker makes deploying services incredibly simple.

### k8s
To use container orchestrator platforms like kubernetes to deploy:

#### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grpc-hello
  labels:
    app: grpc-hello
spec:
  replicas: 1
  selector:
    matchLabels:
      name: grpc-hello
  template:
    metadata:
      labels:
        name: grpc-hello
    spec:
      containers:
      - name: grpc-hello
        image: yogin16/grpc-hello:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 10000
```

#### service.yaml

```yaml
kind: Service
apiVersion: v1
metadata:
  name: grpc-hello
  labels:
    app: grpc-hello
spec:
  selector:
    name: grpc-hello
  ports:
  - protocol: TCP
    port: 10000
    targetPort: 10000
    name: grpc
  type: NodePort
```

To deploy in kubernetes cluster run:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Above config is updated in the [git-repo](https://github.com/yogin16/grpc-spring-gradle-example).

### References:
1. https://www.sumologic.com/blog/devops/kubernetes-vs-docker/
1. https://docs.docker.com/engine/reference/commandline/image_build/
1. https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
1. https://kubernetes.io/docs/tutorials/kubernetes-basics/deploy-app/deploy-intro/
1. https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
1. https://hub.docker.com/_/java/