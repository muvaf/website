---
title: "Introducing kr8s, a new Kubernetes client library for Python inspired by kubectl"
date: 2023-06-19T00:00:00+00:00
draft: false
author: "Jacob Tomlinson"
categories:
  - blog
tags:
  - project
  - kubernetes
  - python
  - kr8s
  - kubectl-ng
---

For the last few months I've been tinkering with a new Kubernetes client library for Python called [kr8s](https://github.com/kr8s-org/kr8s). 

[![Kr8s logo](logo-wide.png "https://github.com/kr8s-org/kr8s")](https://github.com/kr8s-org/kr8s)

The project has reached the point where it has enough features and is stable enough that I'm starting to use it in my other projects, so it feels like a good time to announce it to the world.

```python
import kr8s

api = kr8s.api()
pods = api.get("pods", namespace=kr8s.ALL)

for pod in pods:
    print(pod.name)
```

## Why create a new Python Kubernetes library?

There are many [Python Kubernetes libraries](https://kubernetes.io/docs/reference/using-api/client-libraries/#community-maintained-client-libraries) out there, this one is mine. Let me explain a little about why I am building it.

For the last 6 years I’ve maintained [dask-kubernetes](https://github.com/dask/dask-kubernetes), a Python library for deploying [Dask](https://www.dask.org/) clusters on [Kubernetes](https://kubernetes.io/). In that time I’ve tried nearly every [Python Kubernetes client library on PyPI](https://pypi.org/search/?q=kubernetes). In fact dask-kubernetes today uses more than five different libraries and tools to interact with the Kubernetes API. Each one has different strengths and weaknesses, features and bugs. To satisfy all of the needs of dask-kubernetes there is no one library that can do it alone.

I reached a point where I had to ask myself, should I continue to build wrappers and shims in dask-kubernetes to homogenize the various dependencies? Should I contribute to an existing one to fill in the blanks? Or can I build one library to rule them all?

Don't worry, I didn't fall into the trap of [xkcd #927](https://xkcd.com/927/) (hopefully). 

[![xkcd 927](xkcd-927.png "xkcd #927")](https://xkcd.com/927/)

Instead I decided to build exactly the library I needed. Not a perfect universal library to supersede everything, not a wrapper for everything that exists. Just the library I need to solve my use case, to reduce complexity in my projects and to help me learn the things I need to know to maintain these projects into the future. 

But you never know, this library might just solve your use case too!

## What problems does kr8s solve?

When I started building kr8s I had a loose goal of building something that was simple, extensible and complete. The main problem I wanted to solve was to reduce the complexity of maintaining dask-kubernetes. As I've tinkered with it and it's taken shape that has matured into the following mission statement.

> Kr8s is a Python client library for Kubernetes that is simple, extensible and feels familiar for folks who already know how to use kubectl.

Every Kubernetes client library that I've used in the past is modelled after the Kubernetes API. Some are entirely autogenerated from the [Kubernetes API spec](https://github.com/kubernetes/kubernetes/blob/master/api/openapi-spec/swagger.json). When I think about all of the pain points I have with these libraries it all boils down to the complexity of low-level API interactions and unfamiliarity with the API itself.

But when people interact with Kubernetes they use [`kubectl`](https://kubernetes.io/docs/reference/kubectl/), I know `kubectl` pretty well and so do folks who contribute to dask-kubernetes. So maybe that's a better model for implementing a library? When you run `kubectl` commands it simplifies things for you, it makes good assumptions and provides useful abstractions. 

When I run `kubectl get pods` to list the Pods in my current namespace I can use the short name of the resource, I don't need to specify any authentiction info, I don't need to think about which API version Pods are in, I don't need to think about which namespace I currently have selected, I don't need to think about HTTP sessions or anything else. I can be more explicit if I want to, but `kubectl` is designed for humans and helps me as much as it can.

Kr8s has a similar design philosophy. If I want to get a list of Pods in my current namespace with kr8s I can do the following.

```python
import kr8s
api = kr8s.api()
pods = api.get("pods")
```

The [kr8s client API](https://docs.kr8s.org/en/latest/client.html) is analogous to `kubectl`. It has methods you can call to interact with Kubernetes and exposes the core functionality of the API such as listing, creating, updating and deleting resources. But it also takes things further and provides useful utilities which you would expect to find in `kubectl` such as waiting for resource conditions.

Kr8s also has an [object API](https://docs.kr8s.org/en/latest/object.html), but is a little different to other other libraries. Many libraries are designed to have 1:1 representations of Kubernetes object models as Python objects. These objects strictly have the same data structure as specified by the Kubernetes API and the same methods that the API provides. Kr8s objects on the other hand are an extensible, flexible, minimal-set of these resources.

Kr8s has classes to represent the most common Kubernetes resources, but it is by no means a complete set. To resolve this kr8s has a base `kr8s.objects.APIObject` class which anyone can [subclass and implement any resource they like](https://docs.kr8s.org/en/latest/object.html#extending-the-objects-api). All resources have basic CRUD operations but also implement useful utilities available in `kubectl` such as port forwards.

The Kubernetes API provides a method to forward data to a Pod's ports, but leaves it as an exercise to the user to implement code to handle the protocol and transfer data back and forth. The [example showing a port forward from the official `kubernetes` Python library](https://github.com/kubernetes-client/python/blob/master/examples/pod_portforward.py) is over 200 lines long.

Kr8s aims to be a "batteries included" library and implementings things like port forwarding for you.

```python
from kr8s.objects import Pod

pod = Pod.get("nginx-pod")

with pod.portforward(80) as port:
    # Interact with nginx via local `port`
```

Other important distinctions are around authentication, API objects and HTTP sessions. Kr8s endeavours to abstract these away by default and make sensible assumptions on your behalf. For example we can create a Pod without even creating an API object.

```python
from kr8s.object import Pod

pod = Pod({
        "apiVersion": "v1",
        "kind": "Pod",
        "metadata": {
            "name": "my-pod",
        },
        "spec": {
            "containers": [{
                "name": "pause", 
                "image": "gcr.io/google_containers/pause",
            }]
        },
    })

pod.create()
```

The goal is to match the simplicity of creating a resource with `kubectl`.

```console
$ kubectl create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: pause
    image: gcr.io/google_containers/pause
EOF
pod/my-pod created
```

With `kubectl` I don't have to specify any authenitcation info, or a namespace, or anything like that. It looks everything up from my environment and `kubeconfig`, so kr8s does the same.

For more examples and information about the problems kr8s solves [check out the documentation](https://docs.kr8s.org/en/latest/index.html).

## Is kr8s finished?

Like every side-project kr8s will probably never be finished. But the project has reached the point where the API is relatively stable and the foundations are complete. I have implemented every feature that I need for dask-kubernetes so I have started to swap out the other dependencies in favour of kr8s. 

I'm also confident that if a feature is missing then kr8s exposes enough scaffolding to implement it yourself. This is basically what the other libraries have you do anyway. If someone wants to do the equivalent of `kubectl exec` with kr8s today they will need to make a [low-level API call](https://docs.kr8s.org/en/latest/client.html#low-level-api-calls), but the important thing is they can! 

Kr8s is "good enough" for me to use in dask-kubernetes, so it should be ready for others to start exploring too.

## What is next for kr8s?

I'm actively developing kr8s. While the basic functionality is done there is so much scope for higher-level abstractions and syntactic sugar to make the experience of developing for Kubernetes in Python feel as smooth as using `kubectl` on the command line.

When building out a library for other developers to use I always like to use the library myself to make sure I understand the pain points that others might come across. While dask-kubernetes is a great project for me to do this with it also only uses a subset of the Kubernetes API. In order for kr8s to be useful for as many people as possible I need to keep playing with more and more of the API.

To resolve this I'm also tinkering with another side-project called [`kubectl-ng`](https://github.com/kr8s-org/kr8s/tree/main/examples/kubectl-ng). This is a reimplementation of `kubectl` in Python using `kr8s`, [`rich`](https://rich.readthedocs.io/en/stable/introduction.html) and [`typer`](https://typer.tiangolo.com/).

`kubectl-ng` is much more of a toy project and isn't really ready yet for others to use, and maybe never will be. But by reimplementing `kubectl` I can understand more deeply what is abstracted away, what API calls `kubectl` makes under the hood and how that is exposed to the user. By building `kubectl-ng` I can find great bits of functionality that I can push up into the kr8s API. I'll probably write a follow-up post about my plan for `kubectl-ng`.

So my focus for the next few months will be to continue to build out `kubectl-ng` to drive fixes and enhancements in kr8s along the way.

## Wrap up

I am at the point where kr8s is ready for folks to use, but still has a long way to go. I intend to post about my journey building kr8s so if you're interested in following along then [subscribe to my blog in your favourite RSS reader](https://jacobtomlinson.dev/feed.xml) or [follow me on twitter](https://twitter.com/_jacobtomlinson).