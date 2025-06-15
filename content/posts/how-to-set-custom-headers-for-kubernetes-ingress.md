---
title: "How to set custom headers for Kubernetes Ingress"
date: 2025-6-15
author: "Furkan Demir"
tags: ["Kubernetes", "Ingress-Nginx"]
categories: ["Kubernetes"]
draft: false
---

You may need to set some specifications to forward custom headers, if you are using stock ingress controller for your cluster.

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: "ingress-nginx"
    app.kubernetes.io/instance: "ingress-nginx"
    app.kubernetes.io/component: "controller"
    app.kubernetes.io/part-of: "ingress-nginx"
    app.kubernetes.io/version: "1.12.3"
data:
  use-forwarded-headers: "true"
  proxy-real-ip-cidr: "10.0.0.0/8" # Replace with your trusted CIDR
  compute-full-forwarded-for: "true"
  enable-real-ip: "true"
  allow-snippet-annotations: "true"
```

> ## [use-forwarded-headers](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#use-forwarded-headers)
> 
> If true, NGINX passes the incoming `X-Forwarded-*` headers to upstreams. Use this option when NGINX is behind another L7 proxy / load balancer that is setting these headers.
> 
> If false, NGINX ignores incoming `X-Forwarded-*` headers, filling them with the request information it sees. Use this option if NGINX is exposed directly to the internet, or it's behind a L3/packet-based load balancer that doesn't alter the source IP in the packets

Adding `use-forwarded-headers: "true"` is enough to define custom headers in your `ingress-nginx ConfigMap`. But if you've installed ingress-nginx using the helm chart, you can modify controller's config in the `values.yaml`:

```yml
controller:  
  config:    
    use-forwarded-headers: "true"
    proxy-real-ip-cidr: "10.0.0.0/8" # Replace with your trusted CIDR
    compute-full-forwarded-for: "true"
    enable-real-ip: "true"
    allow-snippet-annotations: "true"
```
