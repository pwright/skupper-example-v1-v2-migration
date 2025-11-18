# Skupper Hello World using YAML

[![main](https://github.com/pwright/skupper-example-v1-v2-migration.git/actions/workflows/main.yaml/badge.svg)](https://github.com/pwright/skupper-example-v1-v2-migration.git/actions/workflows/main.yaml)

#### A minimal HTTP application deployed across Kubernetes clusters using Skupper

This example is part of a [suite of examples][examples] showing the
different ways you can use [Skupper][website] to connect services
across cloud providers, data centers, and edge sites.

[website]: https://skupper.io/
[examples]: https://skupper.io/examples/index.html

#### Contents

* [Overview](#overview)
* [Prerequisites](#prerequisites)
* [Step 1: Install Skupper in your clusters](#step-1-install-skupper-in-your-clusters)
* [Step 2: Set up your namespaces](#step-2-set-up-your-namespaces)
* [Step 3: Apply your YAML resources](#step-3-apply-your-yaml-resources)
* [Step 4: Link your sites](#step-4-link-your-sites)
* [Step 5: Install V2 controller](#step-5-install-v2-controller)
* [Step 6: Create V2 sites.](#step-6-create-v2-sites)
* [Step 7: Link V2 sites.](#step-7-link-v2-sites)
* [Step 8: Expose and consume the backend service.](#step-8-expose-and-consume-the-backend-service)
* [Step 9: Access the frontend](#step-9-access-the-frontend)
* [Cleaning up](#cleaning-up)
* [Summary](#summary)
* [Next steps](#next-steps)
* [About this example](#about-this-example)

## Overview

This example is a variant of [Skupper Hello World][hello-world] v1 that
is deployed using YAML resource definitions instead of imperative
commands.

It contains two services:

* A backend service that exposes an `/api/hello` endpoint in east ns.  It
  returns greetings of the form `Hi, <your-name>.  I am <my-name>
  (<pod-name>)`.

* A frontend service in west ns that sends greetings to the backend and
  fetches new greetings in response.

After this v1 network is up and running, we create a v2 network:

* A v2 site in the east-v2 ns with a connector to backend
* A v2 site in the west-v2 ns with a new frontend deployment

This avoids clashes with the existing sites.

Then we can create an attached connector to the backend service running in east.
* AttachedConnector in east with a reference to the v2 site in east-v2
* AttachedConnectorBinding in east-v2

On the west-v2 namespace, we need to deploy a new frontend service to consume the backend service.

[hello-world]: https://github.com/skupperproject/skupper-example-hello-world

## Prerequisites

* The `kubectl` command-line tool, version 1.15 or later
  ([installation guide][install-kubectl])

* Access to at least one Kubernetes cluster, from [any provider you
  choose][kube-providers]

[install-kubectl]: https://kubernetes.io/docs/tasks/tools/install-kubectl/
[kube-providers]: https://skupper.io/start/kubernetes.html

## Step 1: Install Skupper in your clusters

Use the `kubectl apply` command to install the Skupper
controller in each cluster.

_**West:**_

~~~ shell
kubectl apply -f skupper.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f skupper.yaml
namespace/skupper-site-controller created
serviceaccount/skupper-site-controller created
clusterrole.rbac.authorization.k8s.io/skupper-site-controller created
clusterrolebinding.rbac.authorization.k8s.io/skupper-site-controller created
deployment.apps/skupper-site-controller created
~~~

_**East:**_

~~~ shell
kubectl apply -f skupper.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f skupper.yaml
namespace/skupper-site-controller created
serviceaccount/skupper-site-controller created
clusterrole.rbac.authorization.k8s.io/skupper-site-controller created
clusterrolebinding.rbac.authorization.k8s.io/skupper-site-controller created
deployment.apps/skupper-site-controller created
~~~

## Step 2: Set up your namespaces

Skupper is designed for use with multiple Kubernetes namespaces,
usually on different clusters.  The `skupper` and `kubectl`
commands use your [kubeconfig][kubeconfig] and current context to
select the namespace where they operate.

[kubeconfig]: https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/

Your kubeconfig is stored in a file in your home directory.  The
`skupper` and `kubectl` commands use the `KUBECONFIG` environment
variable to locate it.

A single kubeconfig supports only one active context per user.
Since you will be using multiple contexts at once in this
exercise, you need to create distinct kubeconfigs.

For each namespace, open a new terminal window.  In each terminal,
set the `KUBECONFIG` environment variable to a different path and
log in to your cluster.  Then create the namespace you wish to use
and set the namespace on your current context.

**Note:** The login procedure varies by provider.  See the
documentation for yours:

* [Minikube](https://skupper.io/start/minikube.html#cluster-access)
* [Amazon Elastic Kubernetes Service (EKS)](https://skupper.io/start/eks.html#cluster-access)
* [Azure Kubernetes Service (AKS)](https://skupper.io/start/aks.html#cluster-access)
* [Google Kubernetes Engine (GKE)](https://skupper.io/start/gke.html#cluster-access)
* [IBM Kubernetes Service](https://skupper.io/start/ibmks.html#cluster-access)
* [OpenShift](https://skupper.io/start/openshift.html#cluster-access)

_**West:**_

~~~ shell
export KUBECONFIG=~/.kube/config-west
# Enter your provider-specific login command
kubectl create namespace west
kubectl config set-context --current --namespace west
~~~

_**East:**_

~~~ shell
export KUBECONFIG=~/.kube/config-east
# Enter your provider-specific login command
kubectl create namespace east
kubectl config set-context --current --namespace east
~~~

## Step 3: Apply your YAML resources

To configure our example sites and service bindings, we are
using the following resources:

West:

* [site.yaml](west/site.yaml) - Skupper configuration for West
* [frontend.yaml](west/frontend.yaml) - The Hello World frontend

East:

* [site.yaml](east/site.yaml) - Skupper configuration for East
* [backend.yaml](east/backend.yaml) - The Hello World backend

Let's look at some of these resources in more detail.

#### Resources in West

The `site` ConfigMap defines a Skupper site for its associated
Kubernetes namespace.  This is where you set site configuration
options.  See the [config reference][config] for more
information.

[config]: https://skupper.io/docs/yaml/index.html

[site.yaml](west/site.yaml):

~~~ yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: skupper-site
data:
  name: west
~~~

#### Resources in East

Like the one for West, here is the Skupper site definition for
the East.  It includes the `ingress: none` setting since no
ingress is required at this site for the Hello World example.

[site.yaml](east/site.yaml):

~~~ yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: skupper-site
data:
  name: east
  ingress: none
~~~

In East, the `backend` deployment has an annotation named
`skupper.io/proxy` with the value `tcp`.  This tells Skupper to
expose the backend on the Skupper network.  As a consequence,
the frontend in West will be able to see the backend and call
its API.

[backend.yaml](east/backend.yaml):

<pre>apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
  <b>annotations:
    skupper.io/proxy: tcp</b>
spec:
  selector:
    matchLabels:
      app: backend
  replicas: 3
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: quay.io/skupper/hello-world-backend
          ports:
            - containerPort: 8080</pre>

Now we're ready to apply everything.  Use the `kubectl apply`
command with the resource definitions for each site.

**Note:** If you are using Minikube, [you need to start
`minikube tunnel`][minikube-tunnel] before you create the
Skupper sites.

[minikube-tunnel]: https://skupper.io/start/minikube.html#running-minikube-tunnel

_**West:**_

~~~ shell
kubectl apply -f west/site.yaml -f west/frontend.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f west/site.yaml -f west/frontend.yaml
configmap/skupper-site created
deployment.apps/frontend created
~~~

_**East:**_

~~~ shell
kubectl apply -f east/site.yaml -f east/backend.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f east/site.yaml -f east/backend.yaml
configmap/skupper-site created
deployment.apps/backend created
~~~

## Step 4: Link your sites

You can configure sites and service bindings declaratively, but
linking sites is different.  To create a link, you must have the
authentication secret and connection details of the remote site.
Since these cannot be known in advance, linking must be
procedural.

<!--
**Note:** There are several ways to automate the generation and
distribution of tokens across sites, using for example Ansible,
Backstage, or Vault.  See [Token distribution][XXX] for more
information.
-->

This example uses the Skupper command-line tool to generate the
secret token in West and create the link in East.

To install the Skupper command:

~~~ shell
curl https://skupper.io/install.sh | sh
~~~

For more installation options, see [Installing
Skupper][install].

Once the command is installed, use `skupper token create` in
West to generate the token.  Then, use `skupper link create` in
East to create a link.

[install]: https://skupper.io/install/index.html

_**West:**_

~~~ shell
skupper token create ~/secret.token
~~~

_Sample output:_

~~~ console
$ skupper token create ~/secret.token
Token written to ~/secret.token
~~~

_**East:**_

~~~ shell
skupper link create ~/secret.token
~~~

_Sample output:_

~~~ console
$ skupper link create ~/secret.token
Site configured to link to https://10.105.193.154:8081/ed9c37f6-d78a-11ec-a8c7-04421a4c5042 (name=link1)
Check the status of the link using 'skupper link status'.
~~~

If your terminal sessions are on different machines, you may need
to use `scp` or a similar tool to transfer the token securely.  By
default, tokens expire after a single use or 15 minutes after
creation.

## Step 5: Install V2 controller

Use the `kubectl apply` command to install the Skupper V2
controller clusterwide.

_**East:**_

~~~ shell
kubectl apply -f https://skupper.io/install.yaml
~~~

## Step 6: Create V2 sites.

Create version 2 sites using YAML

_**West:**_

~~~ shell
kubectl create ns west-v2
kubectl apply -f west-v2.yaml
~~~

_**East:**_

~~~ shell
kubectl create ns east-v2
kubectl apply -f east-v2.yaml
sleep 30
~~~

## Step 7: Link V2 sites.

Create a token from west and apply it in east.

_**West:**_

~~~ shell
skupper2 token issue token-mig.yaml -n west-v2
~~~

_**East:**_

~~~ shell
skupper2 token redeem token-mig.yaml -n east-v2
~~~

## Step 8: Expose and consume the backend service.

Create an attached connector for backend in east.
Create a listener in new west-v2 namespace

_**East:**_

~~~ shell
kubectl apply -f attached-connector.yaml
kubectl apply -f attached-conn-binding.yaml
~~~

_**West:**_

~~~ shell
kubectl apply -f west/site.yaml -f west/frontend.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f west/site.yaml -f west/frontend.yaml
deployment.apps/frontend created
~~~

## Step 9: Access the frontend

In order to use and test the application, we need external access
to the frontend.

Use `kubectl expose` with `--type LoadBalancer` to open network
access to the frontend service.

Once the frontend is exposed, use `kubectl get service/frontend`
to look up the external IP of the frontend service.  If the
external IP is `<pending>`, try again after a moment.

Once you have the external IP, use `curl` or a similar tool to
request the `/api/health` endpoint at that address.

**Note:** The `<external-ip>` field in the following commands is a
placeholder.  The actual value is an IP address.

_**West:**_

~~~ shell
kubectl expose deployment/frontend --port 8080 --type LoadBalancer
kubectl get service/frontend
curl http://<external-ip>:8080/api/health
~~~

_Sample output:_

~~~ console
$ kubectl expose deployment/frontend --port 8080 --type LoadBalancer
service/frontend exposed

$ kubectl get service/frontend
NAME       TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
frontend   LoadBalancer   10.103.232.28   <external-ip>   8080:30407/TCP   15s

$ curl http://<external-ip>:8080/api/health
OK
~~~

If everything is in order, you can now access the web interface by
navigating to `http://<external-ip>:8080/` in your browser.

## Cleaning up

To remove Skupper and the other resources from this exercise, use
the following commands.

_**West:**_

~~~ shell
kubectl delete -f west/site.yaml -f west/frontend.yaml
kubectl delete -f skupper.yaml
~~~

_**East:**_

~~~ shell
kubectl delete -f east/site.yaml -f east/backend.yaml
kubectl delete -f skupper.yaml
~~~

## Next steps

Check out the other [examples][examples] on the Skupper website.

## About this example

This example was produced using [Skewer][skewer], a library for
documenting and testing Skupper examples.

[skewer]: https://github.com/skupperproject/skewer

Skewer provides utility functions for generating the README and
running the example steps.  Use the `./plano` command in the project
root to see what is available.

To quickly stand up the example using Minikube, try the `./plano demo`
command.
