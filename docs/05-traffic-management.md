# Traffic Management

One of the most powerful features of Istio is to manage the traffic within your service mesh dynamically. In this part of the tutorial, we'll deploy a new version for our service and use it to take a look some of the different traffic management features of Istio.

## Create a new version
Let's modify our existing Hello World app to print a different message as the new version of the service. In [Startup.cs](../src/helloworld-csharp/Startup.cs) change the message to print `v2`:
```csharp
await context.Response.WriteAsync("Hello World v2!");
```

Build and push the Docker image for v2 (replace `meteatamel` with your actual DockerHub): 

```docker
docker build -t meteatamel/istio-helloworld-csharp:v2 .

docker push meteatamel/istio-helloworld-csharp:v2
```

## Deploy the new version
We're now ready to deploy v2 to our Service Mesh. We need to first create a Kubernetes deployment for the new service. Inside the `istio` folder, create [service-v1v2.yaml](../src/helloworld-csharp/istio/service-v1v2.yaml) file:
```yaml
# Copyright 2019 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
apiVersion: v1
kind: Service
metadata:
  name: helloworld-csharp-service
  labels:
    app: helloworld-csharp
spec:
  ports:
  - port: 8080
    name: http
  selector:
    app: helloworld-csharp
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: helloworld-csharp-deployment-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: helloworld-csharp
        version: v1
    spec:
      containers:
      - name: helloworld-csharp
        # Replace meteatamel with your actual DockerHub
        image: meteatamel/istio-helloworld-csharp:v1
        imagePullPolicy: Always #IfNotPresent
        ports:
        - containerPort: 8080
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: helloworld-csharp-deployment-v2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: helloworld-csharp
        version: v2
    spec:
      containers:
      - name: helloworld-csharp
        # Replace meteatamel with your actual DockerHub
        image: meteatamel/istio-helloworld-csharp:v2
        imagePullPolicy: Always #IfNotPresent
        ports:
        - containerPort: 8080
```

This creates a new deployment for `v2`. Let's deploy it:
```bash
$ kubectl apply -f service-v1v2.yaml

service "helloworld-csharp-service" unchanged
deployment.extensions "helloworld-csharp-deployment-v1" unchanged
deployment.extensions "helloworld-csharp-deployment-v2" created
```

Wait for pods to be initialized for `v2`. Once all pods are ready, ping GATEWAY_URL:
```bash
$ curl "http://${GATEWAY_URL}"

Hello World!

$ curl "http://${GATEWAY_URL}"
xs
Hello World v2!
```

Notice how the response is rotating between `v1` and `v2`. This is because both deployments are behind the same Kubernetes service that is exposed behind the Istio Gateway.

## Pin service to a version
What if you want to pin down to a specific version of the service? For example, let's make our service only use `v2` deployment. That's quite easy to do in Istio. You need a couple of things to set this up. First, you need a [DestionationRule](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#DestinationRule) to define what's called a `subset` for each version. Then, you need to update the [VirtualService](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#VirtualService) to point to `v2` subset. 

To create the subsets, create [destinationrule.yaml](../src/helloworld-csharp/istio/destinationrule.yaml) file inside `istio` folder:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: aspnetcore-destinationrule
spec:
  host: aspnetcore-service
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

Notice how the DestinationRule associates Kubernetes labels with subset names. 

Create the DestionationRule:
```bash
$ kubectl apply -f destinationrule.yaml

destinationrule.networking.istio.io "helloworld-csharp-destinationrule" created
```

Finally, let's point the VirtualService to `v2` subset. Create [virtualservice-v2.yaml](../src/helloworld-csharp/istio/virtualservice-v2.yaml) file inside `istio` folder:
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
        subset: v2
```

Update the VirtualService:
```bash
$ kubectl apply -f virtualservice-v2.yaml

virtualservice.networking.istio.io "helloworld-csharp-virtualservice" configured
```

Now, if we now ping the GATEWAY_URL, it will always get a response from `v2`:
```bash
$ curl "http://${GATEWAY_URL}"

Hello World v2!
```

## Traffic splitting
Sometimes, you might want to split traffic between versions for testing. For example, you might want to send 75% of the traffic to the `v1` and 25% of the traffic to the `v2` version of the service. You can easily achieve this with Istio. 

Create a new Create [virtualservice-weights.yaml](../src/helloworld-csharp/istio/virtualservice-weights.yaml) file inside `istio` folder file to refer to the two subsets with different weights:
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
        subset: v1
      weight: 75
    - destination:
        host: helloworld-csharp-service
        subset: v2
      weight: 25
```

Update the VirtualService:
```bash
$ kubectl apply -f virtualservice-weights.yaml

virtualservice.networking.istio.io "helloworld-csharp-virtualservice" configured
```

Now, we'll get a response mostly from `v1` and sometimes from `v2`.

## Fault injection
Another useful development task to do for testing is to inject faults or delays into the traffic and see how services behave in response. 

For example, you might want to return a bad request (HTTP 400) response for 50% of the traffic to `v1`. Create [virtualservice-fault-abort.yaml](../src/helloworld-csharp/istio/virtualservice-fault-abort.yaml) file:
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
  - fault:
      abort:
        percent: 50
        httpStatus: 400
    route:
    - destination:
        host: helloworld-csharp-service
        subset: v1
```

Update the VirtualService:
```bash
$ kubectl apply -f virtualservice-fault-abort.yaml

virtualservice.networking.istio.io "helloworld-csharp-virtualservice" configured
```

Now, we'll get HTTP 400 errors from `v1` half of the time:
```bash
$ curl "http://${GATEWAY_URL}"

fault filter abort
```

Or you might want to add 5 seconds delay to all requests. Create [virtualservice-fault-delay.yaml](../src/helloworld-csharp/istio/virtualservice-fault-delay.yaml) file:
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
  - fault:
      delay:
        fixedDelay: 5s
        percent: 100
    route:
    - destination:
        host: helloworld-csharp-service
        subset: v1
```

Update the VirtualService:
```bash
$ kubectl apply -f virtualservice-fault-delay.yaml

virtualservice.networking.istio.io "helloworld-csharp-virtualservice" configured
```

And you'll see responses delay 5s all the time!

## What's Next?
This concludes our tutorial. In the next step, we'll cleanup resources. 

[Cleanup](06-cleanup.md)