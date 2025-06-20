---
title: "NSEnter and Kubernetes"
date: 2025-06-19
author: "Ugurcan Akkok"
tags: ["Kubernetes", "Linux"]
categories: ["Kubernetes"]
draft: false
---

Today we will deep dive into the linux utility nsenter and we will enter a kubernetes node with it, without using `ssh` or any other middleware.

# Introduction to NSEnter
So, what is nsenter? From `nsenter(1)`:

```text
NAME
       nsenter - run program in different namespaces

SYNOPSIS
       nsenter [options] [program [arguments]]

DESCRIPTION
       The nsenter command executes program in the namespace(s) that are specified in the command-line options (described below). If program is not given, then "${SHELL}" is run (default: /bin/sh).
```

The `nsenter` is a handy tool for *entering* into any given linux namespace. As you might know, a linux namespace is a sandboxing technology used for isolating different aspects of a program. The list of namespaces are as follows, taken from [namespaces(7)](https://man.archlinux.org/man/namespaces.7).

Namespace | Flag            |Page                  |Isolates
----------|-----------------|----------------------|-------------------------------------
Cgroup    | CLONE_NEWCGROUP |cgroup_namespaces(7)  |Cgroup root directory
IPC       | CLONE_NEWIPC    |ipc_namespaces(7)     |System V IPC, POSIX message queues
Network   | CLONE_NEWNET    |network_namespaces(7) |Network devices, stacks, ports, etc.
Mount     | CLONE_NEWNS     |mount_namespaces(7)   |Mount points
PID       | CLONE_NEWPID    |pid_namespaces(7)     |Process IDs
Time      | CLONE_NEWTIME   |time_namespaces(7)    |Boot and monotonic clocks
User      | CLONE_NEWUSER   |user_namespaces(7)    |User and group IDs
UTS       | CLONE_NEWUTS    |uts_namespaces(7)     |Hostname and NIS domain name

As you can see there are numerous aspects you can isolate a process from. The nsenter allows us to *set* our program's namespaces to that of *other* program's.
For example we can set our mount namespace to PID 5912's mount namespace, so we could see the mount points of the PID 5912.
Or we can set our PID namespace to PID 5912's namespace, which would allow us to see other processes in that namespace.
This might seem harmless at first. However, the danger emerges when namespace isolation is actively used for securing different programs on a single host, which is precisely the foundation of Kubernetes.

The usage of `nsenter` is pretty straight-forward. For the basic usage, we can specify that we want all the namespaces `-a` and give the PID `-t <PID>` of the program we want to enter to.

```
$ nsenter --help

Usage:
 nsenter [options] [<program> [<argument>...]]

Run a program with namespaces of other processes.

Options:
 -a, --all              enter all namespaces
 -t, --target <pid>     target process to get namespaces from
 -m, --mount[=<file>]   enter mount namespace
 -u, --uts[=<file>]     enter UTS namespace (hostname etc)
 -i, --ipc[=<file>]     enter System V IPC namespace
 -n, --net[=<file>]     enter network namespace
 -N, --net-socket <fd>  enter socket's network namespace (use with --target)
 -p, --pid[=<file>]     enter pid namespace
 -C, --cgroup[=<file>]  enter cgroup namespace
 -U, --user[=<file>]    enter user namespace
     --user-parent      enter parent user namespace
 -T, --time[=<file>]    enter time namespace
 -S, --setuid[=<uid>]   set uid in entered namespace
 -G, --setgid[=<gid>]   set gid in entered namespace
     --preserve-credentials do not touch uids or gids
     --keep-caps        retain capabilities granted in user namespaces
 -r, --root[=<dir>]     set the root directory
 -w, --wd[=<dir>]       set the working directory
 -W, --wdns <dir>       set the working directory in namespace
 -e, --env              inherit environment variables from target process
 -F, --no-fork          do not fork before exec'ing <program>
 -c, --join-cgroup      join the cgroup of the target process

 -h, --help             display this help
 -V, --version          display version

For more details see nsenter(1).
```

As you can see, `nsenter` provides flexible way to specify the namespace and program. Behind the scenes, it uses the `setns` syscall for entering into given namespace.

# NSEnter and Kubernetes

In Kubernetes, as you might already know, we run separate processes that are isolated from each other on a single host. Those processes could, for example, belong to different customers or different teams. Even if they belong to the same team/people, you want strong isolation because of security. Any given attacker with access to a process (or pod in Kubernetes) should not be able to jump to other processes in the host. We should isolate where we can to improve our security.

# Demo
Let's demo how we can achieve this in Kubernetes! First we create a Kubernetes cluster using [k3d](https://k3d.io/).

```bash
$ k3d create cluster
```

This will create a cluster with a single node. First let's get the name of the node.

```bash
$ kubectl get nodes
NAME                       STATUS   ROLES                  AGE   VERSION
k3d-k3s-default-server-0   Ready    control-plane,master   76m   v1.31.5+k3s1
```

Then we will create a node with `hostPID=true` and `privileged=true`, so that it has necessary permissions for using nsenter command. HostPID allows us to see the processes from the node, so that we can enter the PID 1, which is the same as entering the node. And privileged is required because we need the CAP_SYS_ADMIN capability. From `capabilities(7)`:

```text
       CAP_SYS_ADMIN
              Note: this capability is overloaded; see Notes to kernel developers below.

              (...)
              •  call setns(2) (requires CAP_SYS_ADMIN in the target namespace);
              (...)
```

Note that this capability gives a lot more than setns permission.

Let's create the privileged pod.

```
$ kubectl run -it --privileged=true --overrides='{"spec":{"nodeName":"k3d-k3s-default-server-0","hostPID":true}}' --image alpine nsenter -- sh
If you don't see a command prompt, try pressing enter.
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 /sbin/docker-init -- /bin/k3d-entrypoint.sh server --tls-san 0.0.0.0 --tls-san k3d-k3s-default-serverlb
    7 root      0:00 {k3d-entrypoint.} /bin/sh /bin/k3d-entrypoint.sh server --tls-san 0.0.0.0 --tls-san k3d-k3s-default-serverlb
   87 root      1:14 /bin/k3s server
  127 root      0:09 containerd
(...)
```

We will install the nsenter package.

```
/ # apk add util-linux-misc
(...)
(12/12) Installing util-linux-misc (2.41-r9)
Executing busybox-1.37.0-r18.trigger
OK: 11 MiB in 28 packages
```

Here we can see that, at the moment we are in the container namespace. The filesystem is container's filesystem and hostname is nsenter.

```
/ # ls /
bin    dev    etc    home   lib    media  mnt    opt    proc   root   run    sbin   srv    sys    tmp    usr    var
/ # hostname
nsenter
```

So, let's enter the node!

```
/ # nsenter -a -t 1 /bin/sh
/ # ls /
bin  dev  etc  k3d  lib  output  proc  run  sbin  sys  tmp  usr  var
/ # hostname
k3d-k3s-default-server-0
```

We can even get the kubeconfig from the node!

```
/ # cat /output/kubeconfig.yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: XXXXXXX # redacted
    server: https://127.0.0.1:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    client-certificate-data: XXXXXXX # redacted
    client-key-data: XXXXXXX
```

And we can enter the node in a single command-line.

```
$ kubectl run -it --privileged=true --overrides='{"spec":{"nodeName":"k3d-k3s-default-server-0","hostPID":true}}' --image alpine nsenter -- sh -c 'apk add util-linux-misc && nsenter -a -t 1'
If you don't see a command prompt, try pressing enter.
(8/12) Installing libncursesw (6.5_p20250503-r0)
(9/12) Installing libsmartcols (2.41-r9)
(10/12) Installing skalibs-libs (2.14.4.0-r0)
(11/12) Installing utmps-libs (0.1.3.1-r0)
(12/12) Installing util-linux-misc (2.41-r9)
Executing busybox-1.37.0-r18.trigger
OK: 11 MiB in 28 packages
/ # hostname
k3d-k3s-default-server-0
```

There is also a kubectl plugin for this exact operation, check out [kubectl-node-shell](https://github.com/kvaps/kubectl-node-shell).

So this is how we can use `nsenter` tool to enter a kubernetes node, all without using `ssh` or any other utility.
