---
title: "Cloudflare Tunnel for Kubernetes"
date: 2022-02-05T00:00:00Z
draft: false
---

Cloudflare offers a free to use Tunnel to connect a server e.g. behind a firewall with the cloudflare servers without opening ports. This can be very useful when operating a server at home or in office. To do that there is a client software called 'cloudflared'. It is pretty easy to use that on your local computer to test that out like described [here](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-guide). Running inside a kubernetes cluster is a bit more tricky so in the following I give an example that should help.

## Prepare configuration

First you have to make a local setup to create the credentials and config.yaml according to [this guide](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-guide).

You should now see this files in your ~/.cloudflared/ directory:
![Cloudflared local files](/static/images/cloudflared-local-setup.png)

The config.yaml file should be adjusted:
{{< highlight yaml >}}
url: http://<URL of the kubernetes service>
tunnel: <Leave this you tunnel id>
credentials-file: ~/.cloudflared/credentials.json # here must be exactly this
{{< / highlight >}}

## Kubernetes configuration

The next step is to store the file contents in a kubernetes secret so that the cloudflared deployment can read it from there (we use a secret because cert.pem and credentials.json should be treated securely). To read the file contents as base64 string you can use the following command:
{{< highlight bash >}}
$ base64 -in ~/.cloudflared/cert.pem 
{{< / highlight >}}
{{< highlight yaml >}}
apiVersion: v1
kind: Secret
metadata:
  name: configuration
type: Opaque
data:
  credentials.json: >-
    eyJBY2NvdW50VGFnIjoiMTU0YzE5Yz...
  cert.pem: >-
    LS0tLS1CRUdJTiBQUklWQVRFIEtFWS...
  config.yaml: >-
    dXJsOiBodHRwOi8vZW1pc3NhcnktaW...
{{< / highlight >}}

## Kuberentes deployment

Finally we deploy the cloudflared as a deployment:
{{< highlight yaml >}}
   
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflare-tunnel
  labels:
    app: cloudflare-tunnel
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudflare-tunnel
  template:
    metadata:
      labels:
        app: cloudflare-tunnel
    spec:
      volumes:
      - name: configuration
        secret:
          secretName: configuration
      containers:
      - name: cloudflare-tunnel
        image: cloudflare/cloudflared:2022.1.3
        args:
        - "tunnel"
        - "--no-autoupdate"
        - "run"
        volumeMounts:
        - name: configuration
          mountPath: "~/.cloudflared/"
          readOnly: true
{{< / highlight >}}

We do not need a service or other kubernetes resources, because cloudflared has only outgoing tcp connections inside the kubernetes cluster.