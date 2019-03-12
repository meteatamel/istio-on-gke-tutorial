# Hello World Istio
In this step, we'll create a Hello World app and get its traffic managed by Istio. 

We'll use C# and ASP.NET Core for the app but you can choose any language and framework you're comfortable with. We're assuming that you already have .NET SDK is installed in your system and you have `dotnet` command line tool available.

## Create an app
Let's start with creating an empty ASP.NET Core app:
```bash
dotnet new web -o helloworld-csharp
```

Test that the app works fine locally. Inside the `helloworld-csharp` folder, run the app:
```bash
dotnet run --urls=http://localhost:8080

Hosting environment: Development
Content root path: /Users/atamel/dev/local/test/helloworld-csharp2
Now listening on: http://localhost:8080
Application started. Press Ctrl+C to shut down.
```
You can check in the browser that the url `http://localhost:8080` simply returns `Hello World!`.

## Build and push a Docker image
Before we can deploy the app to Kubernetes, we need to create a Docker image and push to a public container registry like DockerHub or Google Container Registry. We'll use DockerHub. 

Create a [Dockerfile](../src/helloworld-csharp/Dockerfile) for the image:
```
FROM microsoft/dotnet:2.2-sdk

WORKDIR /app
COPY *.csproj .
RUN dotnet restore

COPY . .

RUN dotnet publish -c Release -o out

ENV PORT 8080

ENV ASPNETCORE_URLS http://*:${PORT}

CMD ["dotnet", "out/helloworld-csharp.dll"]
```

Build and push the Docker image (replace `meteatamel` with your actual DockerHub): 

```docker
docker build -t meteatamel/istio-helloworld-csharp:v1 .

docker push meteatamel/istio-helloworld-csharp:v1
```

## Create Kubernetes Service and Deployment
We're now ready to deploy our app to Kubernetes. We need to create a Deployment to run the container in a pod and a Service to expose the pod to the outside world. 

Create an `istio` folder and in that folder, create [service-v1.yaml](../src/helloworld-csharp/istio/service-v1.yaml) file:
```yaml
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
``` 

Create the Deployment and Service:
```bash
kubectl apply -f service-v1.yaml

service "helloworld-csharp-service" created
deployment.extensions "helloworld-csharp-deployment-v1" created
```

Check that Deployment and Service is created:
```bash
kubectl get deployment,svc

NAME                                                    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE
deployment.extensions/helloworld-csharp-deployment-v1   1         1         1            1

NAME                                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)
service/helloworld-csharp-service   ClusterIP   10.23.243.14   <none>        8080/TCP
```

## What's Next?
At this point, you might be tempted to test the app. Even though the app is deployed to Kubernetes, its traffic is managed by Istio. We need to do more work to make our app publicly accessible and that's next section.

[Gateway and VirtualService](03-gatewayvirtualservice.md)