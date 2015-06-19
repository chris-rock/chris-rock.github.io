---
title: Try Kubernetes with Vagrant
author: chris
date: 2015-06-01
template: article.jade
---

To get familiar with kubernetes, it is always good to start with an example. This blog post will setup nginx running on kubernetes.

## Prerequisites aka setup the cluster

Before we are able to start, we need to download kubernetes and install the command line. To prepare the setup:

* install [Vagrant](http://www.vagrantup.com/downloads.html) (>1.6.2)
* install [VirtualBox](https://www.virtualbox.org/wiki/Downloads)

Then clone the git repository and set the provider for kubernetes setup scripts 

```bash
git clone https://github.com/GoogleCloudPlatform/kubernetes.git
export KUBERNETES_PROVIDER=vagrant
cd kubernetes
```

Now, we are ready to start the Vagrant cluster. With `kube-up.sh` vagrant will provision each machine in the cluster. All necessary components to run kubernetes are installed automatically. By default, each VM is running Fedora as base OS.

```bash
$ ./cluster/kube-up.sh
Starting cluster using provider: vagrant
... calling verify-prereqs
... calling kube-up

...

Wrote config for vagrant to /Users/chris/.kube/config
Each machine instance has been created/updated.
  Now waiting for the Salt provisioning process to complete on each machine.
  This can take some time based on your network, disk, and cpu speed.
  It is possible for an error to occur during Salt provision of cluster and this could loop forever.
Validating master
Validating minion-1
.....
Waiting for each minion to be registered with cloud provider

...

Kubernetes cluster is running.  The master is running at:

  https://10.245.1.2

The user name and password to use is located in ~/.kubernetes_vagrant_auth.

... calling validate-cluster
Found 1 nodes.
     1  NAME         LABELS    STATUS
     2  10.245.1.3   <none>    Ready
Validate output:
NAME                 STATUS    MESSAGE   ERROR
controller-manager   Healthy   ok        nil
scheduler            Healthy   ok        nil
etcd-0               Healthy   {"action":"get","node":{"dir":true,"nodes":[{"key":"/registry","dir":true,"modifiedIndex":3,"createdIndex":3}]}}
                     nil
Cluster validation succeeded
Done, listing cluster services:

Kubernetes master is running at https://10.245.1.2
kube-dns is running at https://10.245.1.2/api/v1beta3/proxy/namespaces/default/services/kube-dns

```

Verify that kubernetes is running:

```bash
$ curl --user vagrant:vagrant --insecure https://10.245.1.2/ 
{
  "paths": [
    "/api",
    "/api/v1beta1",
    "/api/v1beta2",
    "/api/v1beta3",
    "/healthz",
    "/healthz/ping",
    "/logs/",
    "/metrics",
    "/static/",
    "/swagger-ui/",
    "/swaggerapi/",
    "/validate",
    "/version"
  ]
}
```
You could also be able to ssh into master and minion:

```bash
$ vagrant ssh master
$ vagrant ssh minion-1
```

Further details are available at [Kubernetes Vagrant Documentation](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/getting-started-guides/vagrant.md)

## Do some magic with Kubernetes CLI

You could use the `cluster/kubectl.sh`, but I find it much easier to install the `kubectl` binary. This can be easily done via:

```bash
# use mac binary
$ wget https://storage.googleapis.com/kubernetes-release/release/v0.17.0/bin/darwin/amd64/kubectl -O /usr/local/bin/kubectl
$ chmod +x /usr/local/bin/kubectl
$ kubectl version

```

Now, we have the `kubectl` command ready for use.

## Start Nginx cluster without pod definition

This first example uses the docker nginx image and runs it as a cluster. It uses a replica of 3 during bootstrap. Afterwards we change the replica to 2 containers. Finally we destroy all running containers.

```bash
# Create new nginx cluster
$ kubectl run-container webserver --image=nginx --replicas=3 --port=80
CONTROLLER   CONTAINER(S)   IMAGE(S)   SELECTOR                  REPLICAS
webserver    webserver      nginx      run-container=webserver   3

# Show all running pods
$ kubectl get pods -l run-container=webserver
POD               IP            CONTAINER(S)   IMAGE(S)   HOST                    LABELS                    STATUS    CREATED      MESSAGE
webserver-6de4a   10.246.1.17                             10.245.1.3/10.245.1.3   run-container=webserver   Running   27 seconds   
                                webserver      nginx                                                        Running   25 seconds   
webserver-koxmk   10.246.1.19                             10.245.1.3/10.245.1.3   run-container=webserver   Running   27 seconds   
                                webserver      nginx                                                        Running   24 seconds   
webserver-lt0e9   10.246.1.18                             10.245.1.3/10.245.1.3   run-container=webserver   Running   27 seconds   
                                webserver      nginx                                                        Running   24 seconds   

$ kubectl get replicationControllers -l run-container=webserver
CONTROLLER   CONTAINER(S)   IMAGE(S)   SELECTOR                  REPLICAS
webserver    webserver      nginx      run-container=webserver   3

# Resize the replica:
$ kubectl resize rc webserver --replicas=2
resized

$ kubectl get replicationControllers -l run-container=webserver
CONTROLLER   CONTAINER(S)   IMAGE(S)   SELECTOR                  REPLICAS
webserver    webserver      nginx      run-container=webserver   2

# Show all kube nodes, previously `kubectl get minions`
$ kubectl get nodes
NAME         LABELS    STATUS
10.245.1.3   <none>    Ready

# Delete running nginx pod with all replicas and services
$ kubectl stop pods,services -l run-container=webserver
$ kubectl delete pods,services -l run-container=webserver
$ kubectl resize rc webserver --replicas=2
pods/webserver-6de4a
pods/webserver-koxmk

```

## Start Nginx cluster with pods definition

We define a pod definition `01-nginx.yaml` to get nginx up and running. The meta data describes the name and labels of the container. This is required to define selectors for replicas, as we use it later. The `containers` define the used docker image. Create the following file:

```yaml
# 01-nginx.yaml
apiVersion: v1beta3
kind: Pod
metadata:
  name: www
  labels: 
    name: www-nginx
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
```

Load the pod into kubernetes.

```bash
# Start nginx pod/container
$ kubectl create -f 01-nginx.yaml 
pods/www

# Check that nginx is running
$ kubectl get pods -l name=www-nginx
```

Now, we adapt the replica by creating a new `ReplicationController` definition:

```yaml
# 01-nginx-replica.yaml
apiVersion: v1beta3
kind: ReplicationController
metadata:
  name: nginx-controller
spec:
  replicas: 2
  # selector identifies the set of Pods that this
  # replication controller is responsible for managing
  selector:
    name: www-nginx
  # podTemplate defines the 'cookie cutter' used for creating
  # new pods when necessary
  template:
    metadata:
      name: www
      labels:
        # Important: these labels need to match the selector above
        # The api server enforces this constraint.
        name: www-nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```

Load the new definition into kubernetes:

```bash
# Increase replica of the current nginx instances
$ kubectl create -f 01-nginx-replica.yaml
replicationcontrollers/nginx-controller

# Check the replica
$ kubectl get replicationControllers -l name=www-nginx         
CONTROLLER         CONTAINER(S)   IMAGE(S)   SELECTOR         REPLICAS
nginx-controller   nginx          nginx      name=www-nginx   2

# Verify the connection to the pods
$ kubectl get pods -l name=www-nginx                                                                                ✘130 
POD                      IP            CONTAINER(S)   IMAGE(S)   HOST                    LABELS           STATUS    CREATED      MESSAGE
nginx-controller-lfeo7   10.246.1.15                             10.245.1.3/10.245.1.3   name=www-nginx   Running   21 minutes   
                                       nginx          nginx                                               Running   21 minutes   
www                      10.246.1.11                             10.245.1.3/10.245.1.3   name=www-nginx   Running   28 minutes   
                                       nginx          nginx 

```

You could easily verify that multiple nginx instances are running:

```bash
$ vagrant ssh minion-1
[vagrant@kubernetes-minion-1 ~]$ curl http://10.246.1.15
[vagrant@kubernetes-minion-1 ~]$ curl http://10.246.1.11
```

The different containers are not useful by itself. We would like to abstract all instances by a service. Create and load the service definition into kubernetes.

```yaml
# 01-nginx-service.yaml
apiVersion: v1beta3
kind: Service
metadata:
  name: nginx-endpoint
  labels: 
    name: www-nginx
spec:
  ports:
    - port: 8000 # the port that this service should serve on
      # the container on each pod to connect to, can be a name
      # (e.g. 'www') or a number (e.g. 80)
      targetPort: 80
      protocol: TCP
  # public ip of minion, only required for vagrant
  # just like the selector in the replication controller,
  # but this time it identifies the set of pods to load balance
  # traffic to.
  selector:
    name: nginx
```

```bash
# Add public endpoint
$ kubectl create -f 01-nginx-service.yaml
services/nginx-endpoint

# Show all running endpoints
$ kubectl get services -l name=www-nginx 
NAME             LABELS           SELECTOR     IP(S)           PORT(S)
nginx-endpoint   name=www-nginx   name=nginx   10.247.212.62   8000/TCP
```

Now we set up a HTTP Health Checks that will restart a the nginx container in case the application crashes.

```yaml
apiVersion: v1beta3
kind: Pod
metadata:
  name: nginx-healthcheck
  labels: 
    name: www-nginx
spec:
  containers:
    - name: nginx
      image: nginx
      # defines the health checking
      livenessProbe:
        # an http probe
        httpGet:
          path: /_status/healthz
          port: 80
        # length of time to wait for a pod to initialize
        # after pod startup, before applying health checking
        initialDelaySeconds: 30
        timeoutSeconds: 1
      ports:
        - containerPort: 80
```

Add the health check to your cluster

```bash
# add a health check
kubectl create -f 01-nginx-health-check.yaml
```

Everything is setup properly. To destory the cluster, delete all definitions from kubernetes:

```
# Delete all added parts
$ kubectl delete -f 01-nginx-service.yaml 
$ kubectl delete -f 01-nginx.yaml
$ kubectl delete -f 01-nginx-health-check.yaml
```

Have fun with containers!

If you have any questions contact me via [Twitter @chri_hartmann](https://twitter.com/chri_hartmann) or [Github](https://github.com/chris-rock)
