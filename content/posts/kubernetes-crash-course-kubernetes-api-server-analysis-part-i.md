---
title: "Kubernetes Crash Course, Kubernetes API Server Analysis, Part I"
date: 2025-07-20
author: "Furkan Demir"
tags: ["Kubernetes", "control-plane", "kube-apiserver"]
categories: ["Kubernetes"]
draft: false
---

Kubernetes, a distributed operating system for the cloud, relies on a sophisticated control plane to achieve its declarative model of infrastructure management. This control plane is not merely a collection of services; it's a tightly integrated system of specialized components that continuously work to reconcile the actual cluster state with the desired state defined by users.

This series offers a rigorous exploration of the Kubernetes control plane, focusing on both its functional architecture and key source code implementations. Objective is to provide an in-depth understanding of how these distributed components interact, manage state, and enforce policies at scale.

Initial focus is the Kubernetes API Server (`kube-apiserver`). Serving as the singular, authoritative RESTful endpoint for the entire cluster, the API Server is the foundational interface for all control plane interactions from kubectl commands and client SDKs to internal component communication (`kube-scheduler`, `kube-controller-manager`, `kubelet`, `etcd`). Will be analyzed its role in:

**API exposure and versioning**: How it presents the Kubernetes API objects and manages schema evolution.

**Authentication, authorization, and admission control**: The multi-layered security mechanisms that govern access and resource mutation.

**State persistence**: Its exclusive interface with etcd for reliable, consistent cluster state storage.

**High availability patterns**: Architectures that ensure robust and resilient operations.

Understanding the `kube-apiserver` is paramount, as it dictates the interaction model and security posture of the entire Kubernetes ecosystem. Join us as we dissect its operational principles and delve into the underlying code that enables unparalleled orchestration capabilities of Kubernetes.

## 1. Introduction to API Server

The **Kubernetes API Server** (`kube-apiserver`) remains the **central control plane endpoint**. It:

- Exposes a **RESTful API** used by all cluster components and external tools.
- Performs **CRUD operations**, including validation, authentication, and authorization.
- Stores cluster state in **etcd**.
- Addresses **HA requirements** via multiple API server instances behind a load balancer.

## 2. API Interface and Access Methods

### Serving Ports

- **Secure Port:** `6443` — Only supported endpoint for HTTPS requests.
- **Insecure Port:** `8080` — Fully **removed** (since v1.20).

### Clients

- `kubectl`, client SDKs (Go, Python, Java, etc.).
- Internal components (scheduler, controllers, kubelet).
- HA setups: Load balancing across API server instances.

## 3. Central Hub for Cluster Communication

The API server is the **sole access point** to etcd:

- All cluster components use it to `GET`, `LIST`, `WATCH`, `POST`, `PATCH`, and `DELETE` resources.
- Caching via informers reduces load on etcd and speeds up response times.

## 4. Cluster Security Mechanisms

### 4.1 Authentication

Supported methods:

- **Client Certificates**
- **Bearer Tokens** (service accounts, OIDC, static tokens — static tokens discouraged)
- **OIDC**
- **Webhook Token Authentication**
- **Bound Service Account Tokens** (default since v1.24): short-lived, audience-bound, auto-rotated

> **Deprecated/removed**: Basic Auth (v1.25+), static token files, Keystone integration.

### 4.2 Authorization

Controlled via `--authorization-mode`. Available modes:

- `AlwaysAllow` / `AlwaysDeny` (testing)
- `RBAC` — **default and recommended**
- `Webhook`
- `Node` — for kubelet/nodes

> **ABAC** is deprecated; production environments should use **RBAC**.

#### RBAC Example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader
  namespace: default
subjects:
- kind: User
  name: alice
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 4.3 Admission Control

Admission plugins are managed via `--enable-admission-plugins`/`--disable-admission-plugins`.

Common plugins in v1.33:

- `NamespaceLifecycle`, `NodeRestriction`
- `ResourceQuota`, `LimitRanger`
- `MutatingAdmissionWebhook`, `ValidatingAdmissionWebhook`
- **`PodSecurity`** (replaces deprecated PodSecurityPolicy)
- `DefaultTolerationSeconds`, `DefaultIngressClass`, etc.

> `PodSecurityPolicy` was removed in v1.25—use `PodSecurity` or external policy engines (OPA, Gatekeeper).

### 4.4 Secrets Management

Secrets provide secure storage of credentials (OAuth tokens, SSH keys, etc.).

Supported types:

- `Opaque`
- `kubernetes.io/service-account-token`
- `kubernetes.io/dockerconfigjson`
- `kubernetes.io/tls`

#### Example `Opaque` Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

### 4.5 Service Accounts

Provide Pod identity to access the API:

- Namespaced and auto-created (with one default SA per namespace)
- Tokens are mounted into Pods under `/var/run/secrets/...`
- Since v1.24, Pods use **Bound Service Account Tokens** (default, secure, short-lived)

## 5. What's New in v1.33 ("Octarine")

Release **v1.33.0**, officially shipped on **April 23, 2025**, introduces:

- **64 KEPs**: 14 stable, 17 beta, 16 alpha enhancements
- Improvements in security, scalability, and usability
- Ongoing deprecations and feature cleanups consistent with recent release cycles

## 6. Changes Since v1.3

| Feature                            | Status in v1.33.0                            |
|------------------------------------|----------------------------------------------|
| Insecure HTTP port (`8080`)        | **Removed** since v1.20                      |
| ABAC authorization                 | **Deprecated**; remove in practice           |
| Basic Auth / static token files    | **Removed** since v1.25                      |
| PodSecurityPolicy (PSP)            | **Removed** since v1.25                      |
| Bound service account tokens       | **Default** since v1.24                      |
| PodSecurity admission plugin       | **Standard mechanism** since PSP removal     |
| Server-side Apply                  | **Stable** since v1.22                       |
| `kubectl debug`                    | Supports ephemeral containers                |

---

**References:**

- [v1.33.0 Release Notes](https://kubernetes.io/blog/2025/04/23/kubernetes-v1-33-release/)
- [https://kubernetes.io](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)