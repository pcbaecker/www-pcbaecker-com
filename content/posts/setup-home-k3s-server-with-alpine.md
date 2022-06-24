---
title: "How to setup a k3s server with alpine linux"
date: 2021-12-31T09:34:33Z
draft: false
---

Attention: This guide does NOT work on the raspberry pi!

I bought the new Macbook Pro with M1 Pro. The power of that device deprecates my Ryzen 3700X and so I decided to use that one as a server. I measured the power consumption with GPU (RX580) attached at about 90W and without at about 50W. So I used the GPU while installation and removed it afterwards. 

As OS I decided to use [Alpine Linux](https://www.alpinelinux.org/) Extended version because it is very lightweight. Flashing and booting the .iso was straight forward. After booting you have to login as root without password and use the following command to install:
{{< highlight bash >}}
setup-alpine
{{< / highlight >}}

After reboot into the OS you have to allow SSH for the root user by changing the config file
{{< highlight bash >}}
vi /etc/ssh/sshd_config
{{< / highlight >}}
Search for the line that says 'PermitRootLogin' and change that to
{{< highlight text >}}
...
PermitRootLogin yes
...
{{< / highlight >}}
after a reboot you should be able to login per SSH.

To install k3s you can use the following command. I decided to disable some of the predefined software in favour for more control.
{{< highlight bash >}}
curl -sfL https://get.k3s.io | sh -s - --disable=traefik --disable=servicelb --disable=metrics-server
{{< / highlight >}}

To test if the kubernetes is up and running ssh into the server and execute:
{{< highlight bash >}}
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
kube-system   local-path-provisioner-64ffb68fd-q5cfn   1/1     Running   0          32m
kube-system   coredns-85cb69466-zvhxd                  1/1     Running   0          32m
{{< / highlight >}}

To configure kubectl or Lens on your local computer you can download the ~/.kube/config with:
{{< highlight bash >}}
cat /etc/rancher/k3s/k3s.yaml
{{< / highlight >}}




