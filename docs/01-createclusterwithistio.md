# Create GKE Cluster with Istio
[Installing Istio on GKE](https://cloud.google.com/istio/docs/istio-on-gke/installing) page describes in detail how to prepare for installation by enabling Kubernetes Engine API, setting a region with `gcloud` and making sure `kubectl` is setup properly. It also talks about installing different versions of Istio and strict or permissive mTLS options. We assume that you already did all the prep work.

## Create cluster
First, let's create a GKE cluster with strict mTLS option:
```bash
$ gcloud beta container clusters create hello-istio \
    --addons=Istio --istio-config=auth=MTLS_STRICT \
    --cluster-version=latest \
    --machine-type=n1-standard-2 \
    --num-nodes=4
```

In the end, you should see something similar to this:
```bash
NAME         LOCATION        MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION 
hello-istio  europe-west4-a  1.12.5-gke.5    35.204.127.85  n1-standard-2  1.12.5-gke.5
```

## Create cluster rolebinding
Next, grant cluster admin permissions to the current user. You need these permissions to create the necessary role based access control (RBAC) rules for Istio:
```bash
$ kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value core/account)

clusterrolebinding.rbac.authorization.k8s.io "cluster-admin-binding" created
```

## Verify installation
At this point, Istio should be up and running. Let's check Istio pods first. They should all be in `Running` or `Completed` states:
```bash
$ kubectl get pod -n istio-system

NAME                                      READY     STATUS      RESTARTS   AGE
istio-citadel-7d7bb58cd7-hst47            1/1       Running     0          11m
istio-cleanup-secrets-vmnpx               0/1       Completed   0          11m
istio-egressgateway-764d46c6d5-krsvc      1/1       Running     0          11m
istio-galley-845d5d596-d4ff7              1/1       Running     0          11m
istio-ingressgateway-5b7bf67c9b-9dqmj     1/1       Running     0          11m
istio-pilot-668bf94f44-n9zfc              2/2       Running     0          11m
istio-policy-556ff56f5c-d7mnn             2/2       Running     0          11m
istio-sidecar-injector-65797b8bcd-sppx9   1/1       Running     0          11m
istio-telemetry-57464b9744-vc69l          2/2       Running     0          11m
promsd-8cc5d455b-p9krv                    2/2       Running     1          11m
```
We can also see Istio services:
```bash
kubectl get svc -n istio-system
NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                                        
istio-citadel            ClusterIP      10.23.245.52    <none>          8060/TCP,9093/TCP           
istio-egressgateway      ClusterIP      10.23.246.50    <none>          80/TCP,443/TCP
istio-galley             ClusterIP      10.23.246.201   <none>          443/TCP,9093/TCP
istio-ingressgateway     LoadBalancer   10.23.248.25    35.20X.XY.XYZ   80:31380/TCP,443:31390/TCP
istio-pilot              ClusterIP      10.23.240.177   <none>          15010/TCP,15011/TCP,8080/TCP,9093/TCP
istio-policy             ClusterIP      10.23.252.237   <none>          9091/TCP,15004/TCP,9093/TCP
istio-sidecar-injector   ClusterIP      10.23.255.94    <none>          443/TCP
istio-telemetry          ClusterIP      10.23.244.174   <none>          9091/TCP,15004/TCP,9093/TCP,42422/TCP
promsd                   ClusterIP      10.23.249.84    <none>          9090/TCP
```

## Enable auto sidecar injection
To let Istio manage your pods, your pods need to have an Envoy sidecar proxy. You can inject these manually but it's easier to inject them automatically by labelling your namespaces. 

Let's enable auto sidecar injection in default namespace:
```bash
$ kubectl label namespace default istio-injection=enabled

namespace "default" labeled
```

Quick check to see if it's enabled:
```bash
$ kubectl get namespace -L istio-injection

NAME           STATUS    AGE       ISTIO-INJECTION
default        Active    23m       enabled
istio-system   Active    23m       disabled
kube-public    Active    23m
kube-system    Active    23m
```

## What's Next?
We're done setting up Istio on GKE. Next, we'll deploy an app and see how its traffic is managed by Istio.

[Hello World Istio](02-helloworldistio.md)