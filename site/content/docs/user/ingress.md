---
title: "Ingress"
menu:
  main:
    parent: "user"
    identifier: "user-ingress"
    weight: 3
---

# Ingress

This guide covers setting up [ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) 
on a kind cluster.

## Setting Up An Ingress Controller

We can leverage KIND's `extraPortMapping` config option when 
creating a cluster to forward ports from the host 
to an ingress controller running on a node. 

We can also setup a custom node label by using `node-labels` 
in the kubeadm `InitConfiguration`, to be used
by the ingress controller `nodeSelector`.


The following ingress controllers are known to work:

 - [Ingress NGINX](#ingress-nginx)

### Ingress NGINX

Create a kind cluster with `extraPortMappings` and `node-labels`.

{{< codeFromInline lang="bash" >}}
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    apiVersion: kubeadm.k8s.io/v1beta2
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
        authorization-mode: "AlwaysAllow"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
  - containerPort: 443
    hostPort: 443
EOF
{{< /codeFromInline >}}

Apply the [mandatory ingress-nginx components](https://kubernetes.github.io/ingress-nginx/deploy/#prerequisite-generic-deployment-command).

{{< codeFromInline lang="bash" >}}
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
{{< /codeFromInline >}}

Apply kind specific patches to forward the hostPorts to the 
ingress controller, set taint tolerations and 
schedule it to the custom labelled node.

```json
{{% readFile "static/examples/ingress/nginx/patch.json" %}}
```

Apply it by running:

{{< codeFromInline lang="bash" >}}
kubectl patch deployments -n ingress-nginx nginx-ingress-controller -p '{{< minify file="static/examples/ingress/nginx/patch.json" >}}' 
{{< /codeFromInline >}}


Now the Ingress is all setup to be used. 
Refer [Using Ingress](#using-ingress) for a basic example usage.

## Using Ingress

The following example creates simple http-echo services 
and an Ingress object to route to these services.

```yaml
{{% readFile "static/examples/ingress/usage.yaml" %}}
```

Apply the contents

{{< codeFromInline lang="bash" >}}
kubectl apply -f {{< absURL "examples/ingress/usage.yaml" >}}
{{< /codeFromInline >}}

Now verify that the ingress works

{{< codeFromInline lang="bash" >}}
# should output "foo"
curl localhost/foo
# should output "bar"
curl localhost/bar
{{< /codeFromInline >}}
