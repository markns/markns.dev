+++
author = "Mark Nuttall-Smith"
title = "Anatomy of a domain-specific PaaS - Kubernetes, Istio and Auth0"
date = "2022-08-30"
description = "How to build a micro PaaS using Kubernetes and Istio"
tags = [
"architecture",
"kubernetes",
"istio",
"PaaS",
]
draft = false
+++

In [part 1]({{< relref "/posts/domain-specific-paas/what-is-a-domain-specific-paas" >}}) of this series we discussed why and when a domain-specific PaaS can be useful.
[Part 2]({{< relref "/posts/domain-specific-paas/domain-specific-paas-a-use-case.md" >}}) continued with the example use case and presented the platform at a high level.
In this post, we'll get into the technical details of how we might implement a domain-specific PaaS using Kubernetes, Istio and Auth0.

As time allows, I'll update this post with links to GitHub repositories containing implementations of the various components.

A word of warning, Kubernetes and Istio are gigantic and complex systems, and this post could never cover all the details from first principles.
Some familiarity with cloud-native technologies is assumed.
Let's dive in.

Take a look again at the snippet of code in part 2. 
Put simply, our task is to provide a system that will take a hunk of code like this, and turn it into a running process, accessible to authorized client applications.
We want the user to be able to edit the code in a browser, and when they save their work it will be pushed to the system.
The system should then turn the code into a Docker image, and deploy it on our Kubernetes cluster.

You may say, but that's crazy, what about version control, CI/CD, unit-testing, etc, and it would be entirely fair to raise these concerns.
However, remember our goal is to make the system _as accessible as possible_ for users, and let's face it Git is not the most beginner-friendly tool.

Naturally, we could roll our own web server to handle our requests from a client, but we'd end up writing a ton of code we can get for free by extending the Kubernetes API via [Custom Resource Definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) (CRDs).

Our example platform is called AeroGrid, and the users are creating AeroGrid "apps" within the platform, so we'll start by defining a new `app.aerogrid.io` CRD. 
After registering this CRD, the Kubernetes API server creates a new RESTful resource path allowing instances of that resource to tbe created, updated and deleted with the usual HTTP requests.
The Kubernetes API also allows you to _watch_ cluster resources (including custom resources), with long running GET requests.
This enables a client to be notified as soon as there is a change in the state of a resource.

Another very nice feature of the API generated for the CRD is that it is fully documented by an OpenAPI specification.
This means we can use [code generation tools](https://github.com/OpenAPITools/openapi-generator) to generate idiomatic, strongly typed client libraries in a variety of languages.
For our example use case we want to generate TypeScript and dotnet clients for use in a web console and in the Excel add-in.

So, what fields do we need in our new `app.aerogrid.io` CRD? 
All Kubernetes resources must have a [metadata](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#metadata) property containing name, namespace and UID fields, so that is taken care of.
Most obviously, we also need somewhere to put our code, so let's add a list of files to the CRD spec.
In most cases, an app will just have an `app.py` and a `requirements.txt`, but a list of files is extensible.
We also want to define the states that an app can be in, as an enum in the Status.

# Hello Operator

The CRD API gives us a way of managing the state of the apps we *want* running in the system - our _desired state_, but we still need a way of actually having that desire reflected by reality.
We need to implement the behaviours required to transform our desires into reality ✨.
This is the work of [Kubernetes operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).

ℹ️ A quick aside - there's [a lot of overlap](https://github.com/kubeflow/training-operator/issues/300) between the CoreOS "operator" terminology and the native Kubernetes "controller" terminology.
Putting it very briefly, an operator is a domain-specific controller.

Taking inspiration from the world of robotics, a Kubernetes operator observes the desired state of a resource, and performs actions to bring reality in line with desire.
This is the principle of the robotic control loop.
The benefit is that distributed systems are unreliable, subject to Byzantine failure, and generally very difficult to reason about.
(TODO: more on why the operator pattern is useful for distributed systems)

There are SDKs available that make it quite straightforward to developer Operators in several languages.
For example, [Golang](https://sdk.operatorframework.io/), [Java](https://javaoperatorsdk.io/) and [Python](https://github.com/nolar/kopf).

When we create a new `app.aerogrid.io` resource, we ultimately want a new container to be run with our code and dependencies installed within.
[`Pods`](https://kubernetes.io/docs/concepts/workloads/pods/) are the basic unit of deployment in Kubernetes, but it's not recommended to create pods directly.
Rather we should use a `Deployment` resource, which can be used to define rollout, scaling and rollback characteristics.

When a user creates a new app in the platform, our app operator will be notified, and will be responsible for creating a new deployment resource.
In turn, the deployment controller will be notified of the new deployment resource, and will create a new `ReplicaSet` resource.
The replicaset controller will then create a number of pod resources (according to the `spec.replicas` config in the deployment).
Next, the `kube-scheduler` component will see that pods have been created without an assigned `Node` and wll select an appropriate node for them to run on.
Finally, the `kubelet` agents running on the involved nodes will actually start the required containers.


# Maintaining the right image at all times

"But wait!", I hear you say, don't we need to have an image to run in the first place?
We do indeed, and because we want to allow our users to install packages using `requirements.txt`, we can't just use the same image for all the apps.
We need to be able to build an image in the cluster if required.

So, before creating the deployment resource, the app operator will create an custom `Image` resource (let's assume we registered it previously), copying the files into it by means of a `ConfigMap` resource.
The image controller will determine if a suitable image exists in our docker registry, or if an image build process needs to be started.
If a build is required, the image controller will create a `Job` resource configured to run a [Kaniko](https://github.com/GoogleContainerTools/kaniko) container.
Kaniko is a tool to build container images from a Dockerfile, inside a container or Kubernetes cluster.
Once the Kaniko job is complete, the new image is pushed to the registry, our image controller is notified, and the new image tag can be updated on the app spec, causing the app controller to run through its control loop again.

# Get connected

Pods running the right image on a node within our cluster is a big step in the right direction.
But pods can be stopped and rescheduled to other nodes for any number of reasons, and the IP they are assigned within the cluster is ephemeral.
We need to be route traffic reliably to our workloads, so we need a stable network address.
This is the job of Kubernetes `Service`s.

Services give us a stable way of finding our pods withing the cluster, but we also need traffic from our clients outside the  cluster to be able to reach the pods.
In Kubernetes, this is the responsibility of the `Ingress` resource and the ingress controller.
There are many types of ingress controller, maintained by the Kubernetes project, and by independent vendors.

For our example, however, we'll use Istio as both our ingress controller and service mesh. Istio provides powerful features like traffic management, observability, and security for service-to-service communication.

# Ingress with Istio Gateway and VirtualService

Instead of a standard Kubernetes Ingress, we define an Istio Gateway and VirtualService to route external traffic into our cluster:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: aerogrid-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: aerogrid-app-routing
spec:
  hosts:
  - "apps.aerogrid.example.com"
  gateways:
  - aerogrid-gateway
  http:
  - match:
    - uri:
        prefix: /apps
    rewrite:
      uri: /
    route:
    - destination:
        host: {{ appName }}.apps.svc.cluster.local
        port:
          number: 80
```

This configuration sends requests from `/apps/{appName}` on the public hostname to the corresponding service in the cluster.

# Service-to-Service Security with mTLS

Istio can automatically enable mutual TLS (mTLS) between all service-to-service traffic, ensuring encryption and identity verification:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default-mtls
spec:
  mtls:
    mode: STRICT
```

With this in place, all pods in the mesh will communicate over encrypted channels without any changes to the application code.

# End-User Authentication with Auth0

To secure our public API and identify users, we delegate authentication to Auth0. We register our AeroGrid API in Auth0 and configure a JSON Web Key Set (JWKS) endpoint. Then we apply an Istio `RequestAuthentication` policy to validate incoming JWTs:

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: default
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  jwtRules:
  - issuer: "https://YOUR_AUTH0_DOMAIN/"
    jwksUri: "https://YOUR_AUTH0_DOMAIN/.well-known/jwks.json"
```

Next, we enforce access control with an `AuthorizationPolicy`:

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: aerogrid-authz
  namespace: default
spec:
  selector:
    matchLabels:
      app: aerogrid-api
  action: ALLOW
  rules:
  - from:
    - source:
        requestPrincipals: ["https://YOUR_AUTH0_DOMAIN/*"]
```

This policy ensures that only requests with a valid Auth0 JWT are allowed to reach our app API.

# Observability and Metrics

Istio also gives us fine-grained telemetry out of the box, including metrics (via Prometheus), distributed tracing (via Jaeger), and logging. We can use these tools to monitor app health, performance, and usage.

# Multi-Tenancy and Isolation

By combining Kubernetes namespaces, Istio policies, and Auth0, we can isolate tenants in AeroGrid:

- Each tenant gets its own Kubernetes namespace.
- Istio `PeerAuthentication` and `AuthorizationPolicy` restrict traffic across namespaces.
- Auth0 uses scopes or claims to identify tenant membership.

# Conclusion

In this post, we've seen how Istio can handle ingress, service mesh security, and end-user authentication via Auth0, all without modifying application code. With Kubernetes CRDs and Operators automating deployment, and Istio managing traffic and security, our AeroGrid platform delivers a powerful, easy-to-use domain-specific PaaS.

In the next part of this series, we'll dive deeper into observability and scaling, exploring how to maintain performance as our user base grows.
