# Cleanup
In this last section, we'll delete everything we created. 

## Delete Istio artifacts
Let's start with deleting Istio Gateway, VirtualService, DestionationRules.

```bash
$ kubectl delete -f gateway.yaml

gateway.networking.istio.io "helloworld-csharp-gateway" deleted

$ kubectl delete -f virtualservice.yaml

virtualservice.networking.istio.io "helloworld-csharp-virtualservice" deleted

$ kubectl delete -f destinationrule.yaml

destinationrule.networking.istio.io "helloworld-csharp-destinationrule" deleted
```

## Delete Kubernetes service and deployment
Next, let's delete the Kubernetes resources:

```bash
$ kubectl delete -f service-v1v2.yaml

service "helloworld-csharp-service" deleted
deployment.extensions "helloworld-csharp-deployment-v1" deleted
deployment.extensions "helloworld-csharp-deployment-v2" deleted
```

## Delete cluster
Finally, if you wish, you can also delete the GKE cluster:

```bash
$ gcloud container clusters delete hello-istio
```