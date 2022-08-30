+++
author = "Mark Nuttall-Smith"
title = "Anatomy of a micro PaaS - part 3"
date = "2022-07-14"
description = "How to build a micro PaaS using Kubernetes and Istio"
tags = [
    "architecture",
    "kubernetes",
    "PaaS",
]
draft = true
+++

## Kubernetes all the things 

## How does an Operator work?
Operators work by extending the Kubernetes control plane and API server. 
Operators allows you to define a Custom Controller that watches your application and performs custom tasks based on its state. 
The application you want to watch is usually defined in Kubernetes as a new object: a Custom Resource (CR) that has its own YAML spec and object type that is well understood by the API server. 
That way, you can define any specific criteria in the custom spec to watch out for, and reconcile the instance when it doesn’t match the spec. The way an Operator’s controller reconciles against a spec is very similar to native Kubernetes controllers, though it is using mostly custom components.

----

OpenAPI spec-first design
Code gen
