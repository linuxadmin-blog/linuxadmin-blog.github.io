---
title: "How Helm works with Kubernetes Namespaces"
date: 2024-12-31
author: "Furkan Demir"
tags: ["Kubernetes", "Helm"]
categories: ["Kubernetes"]
draft: false
---

One of the most useful features of Helm is the ability to store state, which allows you not only to expand charts, but also to carefully delete them, clearing away all traces of their presence... However, it’s a good idea to understand exactly how Helm stores the state, which is what we will do in this article.

## Introduction

I was just interested in learning how Helm stores state. Understanding this issue is not difficult and everything is cleared in the documentation. As of Helm 3, state is kept secret by default.

*Note: you can also store state in the old way in configmap, but you must be more careful about the safety of passwords, private keys and other sensitive data, because by default the RBAC restrictions for configmap are much weaker. Don't forget about the backend in the form of a database. This is especially true when the state exceeds the default maximum secret size (1MiB, [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/#restriction-data-size)).*

Helm 3 stores state in namespace secrets by default. If not set otherwise, this will be the default namespace. You can override the namespace using the `namespace` flag or the `HELM_NAMESPACE` environment variable:

```shell
$ helm repo add grafana https://grafana.github.io/helm-charts
$ helm repo update
$ helm upgrade --install loki grafana/loki-stack --namespace loki-stack --create-namespace
```

This is what the secret itself looks like:

```shell
$ kubectl -n loki-stack get secret
NAME                              TYPE                DATA  AGE
sh.helm.release.v1.loki-stack.v1  helm.sh/release.v1  1     3d21h
```

But the devil is in the details, which is only confirmed by the number of reactions to messages in the [Best Practices/Docs Question – Specify Namespace in Chart?](https://github.com/helm/helm/issues/5465). And now about everything in more detail.

### Let's dive into details

First, let's look at options for managing Kubernetes namespaces in the context of the helm. Let's start with the most obvious but erroneous scenarios.

Namespace in Kubernetes is a key tool for isolating cluster objects. You can set it for an object (for example, a pod) as follows:

```yaml
# templates/nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.25.3
    ports:
    - containerPort: 80
```

If you want to give it to the helm in this form, then you also need to create the namespace itself using the same chart:

```yaml
# templates/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx
```

You can go a little further and define namespace as a variable in the values.yaml file:

```yaml
# values.yaml
namespace: nginx
```

And this is how to call it inside an object template:

```yaml
metadata:
  name: nginx
  namespace: {{ .Values.namespace }}
```

This model is an anti-pattern, because potentially you will get a situation where the state and chart objects can be stored in different namespaces (and probably will be stored). Helm will save its state in the `default` namespace unless you run `helm install` with the `--namespace` flag . The key value must match the variable value in the `values.yaml` file .

This is a fundamental difference from kubectl. In the case of kubectl, if your objects have a namespace with the name X , and you pass the `--namespace` flag with the name `Y` to kubectl itself , then kubectl will crash with the error:

```
error: the namespace from the provided object "<namespace_X>" does not match the namespace "<namespace_Y>". You must pass '--namespace=<namespace_X>' to perform this operation.
```

Kubectl does not allow double interpretation of the configuration, but helm does.

### Don't define a namespace

You can go the other way and not define a namespace at all for any chart object, that is, simply ignore the namespace key. In this case, helm will put everything in the default namespace - both objects and a stateful secret. If you set the `--namespace <name>` and `--create-namespace` flags , then helm will create the required namespace and place everything there (objects and secret). It’s convenient and there are even no problems with “splitting” the configuration, when objects and state are in different namespaces. This is the approach that is officially recommended - namespaces should not be explicitly defined anywhere.

However, unclear points remain. Let's say you run the following two commands sequentially:

```shell
$ helm template test helm-test
$ helm template test helm-test --namespace nginx
```

Helm will render all chart objects and display it in the console output. The only caveat is that in both cases the output will be completely the same . That is, helm will simply ignore the namespace option. I would at least like to see a custom namespace if I specify it explicitly (2nd command), and ideally in both cases.

#### Use Built-in Variable

At the chart level there are a number of built-in variables that are available globally. Among them there is also a variable that stores the name of the release namespace, this is Release.Namespace ([Helm – Built-in Objects](https://helm.sh/docs/chart_template_guide/builtin_objects/)). You can use it as follows:

```yaml
metadata:
  name: nginx
  namespace: {{ .Release.Namespace }}
```

This variable will store the namespace name, which you will pass to the helm using the `--namespace` flag . Or it will give the name of the default namespace `default` . In this case, running the two `helm template` commands from the previous chapter will give different results. This is exactly the behavior that I personally expect from the simplest templating engine, which is Helm.

## Conclusion

Setting the `‐‐namespace` flag for `helm install/upgrade` does not in any way control the corresponding namespace parameter in the Kubernetes object metadata. Moreover, if your objects already contain the namespace parameter: `<name_1>`, then helm will create objects in the namespace `<name_1>` , even if it is explicitly set to the `‐‐namespace <name_2>` flag . But he will put the secret with the state in the namespace `<name_2>` .

In such a simple task, you have to take into account the peculiarities of the helm and take care of the correct processing of the namespace in which both objects and a stateful secret must be created. Kubectl implements similar logic much more correctly, allowing you to enjoy all the benefits of the declarative approach.

The helm developers recommend NOT setting namespaces for Kubernetes objects explicitly, but relying on the `--namespace` and `--create-namespace` flags. I believe that it is more correct to set `namespace: {{ .Release.Namespace }}` for all Kubernetes objects for which the namespace parameter is generally applicable. At a minimum, this will give you a clear logic for the operation of the helm template command , and at a maximum, it will protect you from possible configuration splits.