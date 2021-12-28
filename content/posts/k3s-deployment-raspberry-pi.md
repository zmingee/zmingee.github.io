---
title: "K3s Deployment on Raspberry Pis"
description: Deploy K3s on Raspberry Pis running Alpine Linux
date: 2021-05-10
tags: [k3s, k8s, kubernetes, linux]
---


This serves as a short guide to deploying K3s onto Raspberry Pis. It's written
with a target OS of Alpine Linux in mind, but could easily be adapted for other
distributions.


# Configure Alpine Linux

Instructions on install Alpine Linux on a Raspberry Pi are outside the scope of
this article. It's best to either use a cluster of Rasperry Pis that are the
same version/architecture, or to use an Alpine Linux base image that has the
same architecture. Then you don't have to worry about using different images
architectures on each node.

Once Alpine Linux is installed and configured (in sys mode), you'll want to
append the following options to the Linux cmdline at `/boot/cmdline.txt` on
each node:


```
cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
```

After updating the cmdline, reboot the node.

> You may want to consider your naming scheme. An example would be
> `kube-node[0-9]{1,2}.int.example.test`

## Additional Environment Setup

My own environment has a separate node that provides DNS and NFS. I would
recommend having something similar, so you can have DNS resolution at a
minimum. Running `dnsmasq` and `nfs-server` will suffice when properly
configured, which is outside the scope of this article.


# Deploy K3s Master

We will be installing K3s with a single master. I personally have reliability
issues when attempting clustering with Raspberry Pis, but your mileage may
vary. We'll also be deploying K3s without the `metrics-server`, to cut down
on overhead due to the environment constraints.

Select the node you'll be deploying as the K3s master (such as
`kube-node1.int.example.test`), and run the following:

```shell
curl -sfL https://get.k3s.io | \
    INSTALL_K3S_CHANNEL=latest \
    K3S_NODE_NAME="$(hostname -s)" \
    sh -s - \
    --tls-san="$(hostname -f)" \
    --disable metrics-server
```

A breakdown of what this command is doing:

`INSTALL_K3S_CHANNEL=latest`
: Here we select the ``latest`` channel, rather than the default ``stable``

`K3S_NODE_NAME="$(hostname -s)"`
: Here the K3s node name is set to the short hostname, which should be the
default but is set here to be explicit

`--tls-san="$(hostname -f)"`
: Here a SAN is added to the TLS certs that get created for the full FQDN

`--disable metrics-server`

: Here we disable the metrics-server deployment due to resource constraints

Wait for the K3s deployment to complete on the master before continuing. This
may take a few minutes. When the master is ready, you should see output like
this:

```shell
    kube-node1:~# kubectl get nodes
    NAME         STATUS   ROLES                  AGE   VERSION
    kube-node1   Ready    control-plane,master   10m   v1.21.0+k3s1
```

# Deploy K3s Worker(s)

Now we can move onto deploying K3s workers. You can deploy multiple at a time,
but I generally do one at a time just to be safe.

You'll need to copy the node token from the K3s master to join the worker
nodes, located at `/var/lib/rancher/k3s/server/node-token`

Select the node you'll be deploying the K3s worker agent on (such as
`kube-node2.int.example.test`), and run the following:

```shell
    curl -sfL https://get.k3s.io | \
        INSTALL_K3S_CHANNEL=latest \
        K3S_NODE_NAME="$(hostname -s)" \
        K3S_URL="https://kube-node1.int.example.test:6443" \
        K3S_TOKEN="<NODE-TOKEN-FROM-ABOVE>" \
        sh -
```

A breakdown of what this command is doing:

`INSTALL_K3S_CHANNEL=latest`
: Here we select the `latest` channel, rather than the default `stable`

`K3S_NODE_NAME="$(hostname -s)"`
: Here the K3s node name is set to the short hostname, which should be the
default but is set here to be explicit

`K3S_URL="https://kube-node1.int.example.test:6443"`
: Here we set the K3s master to connect to (update to match your environment)

`K3s_URL="<NODE-TOKEN-FROM-ABOVE>"`
: Here we set the K3s node token to join the master. This is located on the
master at `/var/lib/rancher/k3s/server/node-token`

Wait for the K3s deployment to complete before continuing. This may take a few
minutes. When the worker is ready, you should see output like this:

```shell
    kube-node1:~# kubectl get nodes
    NAME         STATUS   ROLES                  AGE   VERSION
    kube-node1   Ready    control-plane,master   20m   v1.21.0+k3s1
    kube-node2   Ready    <none>                 10m   v1.21.0+k3s1
```

Repeat this on all the worker nodes you would like to join the K3s environment.


# Verify Environment

Your K3s environment should now be complete. We can run some basic checks now:

```shell
    kube-node1:~# kubectl cluster-info
    Kubernetes control plane is running at https://127.0.0.1:6443
    CoreDNS is running at
    https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

```shell
    kube-node1:~# kubectl get-nodes
    NAME         STATUS   ROLES                  AGE    VERSION
    kube-node1   Ready    control-plane,master   30m    v1.21.0+k3s1
    kube-node2   Ready    <none>                 20m    v1.21.0+k3s1
    kube-node3   Ready    <none>                 10m    v1.21.0+k3s1
```


# Extras

## Setup Remote Access

Configuring remote access from your workstation is simple. First install
`kubectl` using your distributions package manager. Then, on the K3s master
node, copy the file `/etc/rancher/k3s/k3s.yaml` to `~/.kube/config`. You'll
want to change the `cluster[0].cluster.server` to match the external name of
the master:

```yaml
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority-data: ...
        server: https://kube-node1.int.example.test:6443
      name: default
```

Once you have the configuration file placed, you can use ``kubectl`` on your
local workstation to manage the K3s environment:

```
[workstation]$ kubectl cluster-info
Kubernetes control plane is running at https://kube-node1.int.example.test:6443
CoreDNS is running at
https://kube-node1.int.example.test:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```


## Deploy cert-manager with Lets Encrypt Issuer

Now we can deploy `cert-manager` to automate certificate management. We'll
also configure an Issuer for Lets Encrypt. The following example uses an ACME
HTTP01 challenge, but you can refer to the [cert-manager documentation][1] for
details on using the DNS01 challenge.


The first step is to install the `CustomResourceDefinitions` and
`cert-manager`:

> This link is for `v1.3.1`, and will likely become stale with new releases.
Please refer to the [cert-manager GitHub Releases page][2] for the most up to
date version.


```shell
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.yaml
```


It may take a few minutes for everything to deploy and populate. You can check
the output of `kubectl get all -A` to check for any resources still in the
`Creating` state.

Once it's complete, we can deploy the Issuer for Lets Encrypt. Create a new
manifest file from the following:

```yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt
    spec:
      acme:
        server: https://acme-v02.api.letsencrypt.org/directory
        email: john.dough@example.com
        privateKeySecretRef:
          name: letsencrypt
        solvers:
        - http01:
            ingress:
              class: traefik
```

You'll want to update the `spec.acme.email` value to match your email for
verification purposes. This manifest will create a ClusterIssuer resource that
will issue certificate requests and certificates from Lets Encrypt. This can be
used to add TLS support to Ingresses. For more information on this, please
refer to the [Securing Ingress Resources][3] guide.

# Conclusion

You should now have a K3s environment that's ready for further use. You can
deploy manifests to this environment like you would any other. I'll write
guides in the future on building manifests and deploying open-source software.

If you would like to use Helm, you can review the [Rancher K3s
documentation][4] on the subject.



[1]: https://cert-manager.io/docs/configuration/acme/
[2]: https://github.com/jetstack/cert-manager/releases
[3]: https://cert-manager.io/docs/usage/ingress/
[4]: https://rancher.com/docs/k3s/latest/en/helm/
