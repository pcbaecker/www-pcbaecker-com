---
title: "How to setup a k3os server at home"
date: 2021-12-31T09:34:33Z
draft: true
---

I used the [latest release of k3os](https://github.com/rancher/k3os/releases) with configuration per cloud-init file. My cloud-init file looked like the following:

{{< highlight yaml >}}
ssh_authorized_keys:
  - ssh-rsa AAAAB3N...
hostname: k3os_master
k3os:
  password: test123
{{< / highlight >}}

Generating the ssh-keypair in the local repository with:

{{< highlight bash >}}
ssh-keygen -f ./k3os_master_key
{{< / highlight >}}

{{< highlight bash >}}
brew install http-server
{{< / highlight >}}

{{< highlight bash >}}
ssh -i k3os_master_key rancher@192.168.178.82
{{< / highlight >}}

// TODO For some reason there appears no node. Some config must be wrong