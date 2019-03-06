# Gateway and VirtualService
In the previous step, we deployed our app to Kubernetes. In this step, we'll get our app's traffic managed by Istio by creating Gateway and Virtual Service (TODO: Link). 

Gateway allows external traffic into Service Mesh. It just specifies the protocol (HTTP/HTTPS) and the ports (80/443) that are exposed. 

VirtualService maps the traffic from Gateway to Kubernetes Services inside the ServiceMesh. 

This is a visual representation of the relationship between Gateway, VirtualService, and Kubernetes Service:

TODO: Link

## Create Gateway

Let's start with creating the Gateway. Our app uses HTTP on port 80, so let's expose those with a Gateway. 

Create a [gateway.yaml](../src/helloworld-csharp/istio/gateway.yaml) file:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: helloworld-csharp-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

Create the Gateway:

```bash
$ kubectl apply -f gateway.yaml

TODO
```

Check that Gateway is created:

```bash
$ kubectl get gateway

TODO
```

## Create VirtualService
Next, we need to route the traffic from Gateway to the Kubernetes Service and we'll do that via VirtualService. 

Create a [virtualservice.yaml](../src/helloworld-csharp/istio/virtualservice.yaml) file:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld-csharp-virtualservice
spec:
  hosts:
  - "*"
  gateways:
  - helloworld-csharp-gateway
  http:
  - route:
    - destination:
        host: helloworld-csharp-service
```

Notice how the VirtualService ties together the Gateway and the Kubernetes Service. Create the VirtualService:

```bash
$ kubectl apply -f virtualservice.yaml

TODO
```

Check that VirtualService is created:

```bash
$ kubectl get virtualservice

TODO
```

## Test the app
We can finally test the app. All the traffic in Istio goes through `istio-ingressgateway`. You can find out the IP and port of `istio-ingressgateway`:

```bash
$ kubectl get svc istio-ingressgateway -n istio-system
```
To make it easier, you can get the ingress host and port and set a `GATEWAY_URL` variable to use in our testing:

```bash
$ export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')

$ export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

To see the response, you can open a browser with `GATEWAY_URL` or use curl:

```bash
$ curl -o /dev/null -s -w "%{http_code}\n" "http://${GATEWAY_URL}"

TODO: Show Hello World!
```

## What's Next?
Next, we'll look at some of the services you can add-on to Istio such as Prometheus, Grafana, Jaeger and ServiceGraph.

[Istio add-ons](04-istio-addons.md)