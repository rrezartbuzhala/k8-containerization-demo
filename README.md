# Gettomg Started with Kubernetes

## Prerequisites 
* .NET SDK installed
* Docker Desktop

## Scenario
You have an application and it need to be setup in a way where it's able to be deployed into a Kubernetes cluster.

The application can be found in the `apps` folder, it was generated using the dotnet template command.

```bash
dotnet new webapi -o app1
```

This command uses a native dotnet template `webapi` to create a web API project that generates fake weather data.

To run an application change directory to the app folder using `cd apps/app1`, then execute `dotnet run` to build and launch the application. Using the port noted in the console logs you can append `/swagger` and the Swagger documentation will be loaded.

## Containerization
Containerization is the act of creating portable, isolated, and reproducible environments for applications.

Docker is the most popular tool to do this, however there's also other tools like Containerd. For the purpose of this demo we will use Docker throughout.

Within the context of Docker there are two important questions that we first need to answer. What is a `Dockerfile`? and what is a `docker-compose` file? A `Dockerfile` is used to compile and build a Docker Image, a Docker Image contains everything related to an environment that an application needs to be run. A `docker-compose` file is used to manage one or more images including setting up storage volumes, virtual networks, container resources, and other images that might be a dependency for an application like a database.

### Containerizing our applications

Currently our application is not containerized, first need to create a `Dockerfile` in the root of each project, and paste the following.

```bash
# Use the official Microsoft .NET runtime image as the base image
FROM mcr.microsoft.com/dotnet/aspnet:7.0

# Set the working directory inside the container
WORKDIR /app

# Copy the published output of the application to the container
COPY ./publish /app

# Set the port the application will listen on
ENV ASPNETCORE_URLS=http://+:8080

# Expose the port the application will run on
EXPOSE 8080

# Start the application using the dotnet runtime
ENTRYPOINT ["dotnet", "app1.dll"]
```

Note that in the `ENTRYPOINT` command there is a reference to `app1` in this example. This should be changed to the project name of your application.

In the `COPY` command there is a reference to a local directory called `publish`. This directory does not exist yet, however we can fix that by publishing our application.

```bash
dotnet publish --configuration Release --output ./publish
```

Next we can generate our image.

```bash
docker build --tag app1 .
```

To test our new Docker image we can start a container.

```bash
docker run --rm -p 5000:8080 --name my_app1 app1
```

Loading the url `http://localhost:5000/WeatherForecast` in a browser or executing a cUrl should return a JSON response with a list of fake weather forecasts.

```bash
curl -XGET 'http://localhost:5000/WeatherForecast'
```

Note that by default the Swagger endpoint we used earlier is not enabled on production environments by default. In order to change the environment of your application you can set the environment variable `ASPNETCORE_ENVIRONMENT` to `Development` in the Dockerfile.

If an error comes up that the container is already running it should be stopped. Since we're using the `--rm` command to start the container, the container will be removed automatically.

```bash
docker stop my_app1
```

The final step is to publish our image to a Docker registery. The app use for this Demo is already available as the image `jeanfrg/app1`, however to publish your own we need to use these commands.

```bash
docker tag app1 <your-docker-hub-id>/app1:v1
docker push <your-docker-hub-id>/app1:v1
```

## Kubernetes

Kubernetes is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications -- we call these tools Orchestrators.

### Setting up a Kubernetes local environment

There are multiple ways that one can set up a local kubernetes environment. Since we're using Docker already the easiest way is to use Docker's native integration.

[MiniKube](https://minikube.sigs.k8s.io/docs/start/) is also a good tool that we can use to achieve this.

[Enable and start Kubernetes](https://docs.docker.com/desktop/kubernetes/#enable-kubernetes) on Docker Desktop.

Test if the installation was successful

```bash
kubectl version --short
```

Next we need to create some Kubernetes manifests, which tell Kubernetes how to configure our application.

Create a file called `deployment.yaml` within `kubernetes/apps/app1` with the content

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: jeanfrg/app1:latest
        ports:
        - containerPort: 8080
```

This configuration file defines a deployment with one replica of our Docker image, and exposes port 80 on the container.

Note that if you have published your own docker image, `spec.template.spec.[0].image` should reference your image.

Next we need to create a Kubernetes service which will allow us to expose the service out of Kubernetes so we can hit it locally. Next to the `deployment.yaml` file, create a file called `service.yaml` with the following content.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app1
spec:
  selector:
    app: app1
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 8080
  type: LoadBalancer
```

This configuration file defines a LoadBalancer service that exposes your application on port 80 and routes traffic to the target port 80 on the container.

Change directory to `kubernetes/apps/app1` and apply these configurations.

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Note we can also change the directory to `kubernetes/apps` and use `kubectl apply -f app1` to apply all configurations within the folder without specifying each one.

Checking the configuration status in Kubernetes.

```bash
kubectl get deployments
kubectl get services
```

The information printed out from the services command with show us exactly how we're able to access our application from outside Kubernetes.

```
NAME         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
app1         LoadBalancer   10.101.183.235   localhost     5000:31785/TCP   1m
```

In this example we use the value of the external IP column as the host and the first port specified in the port column.

```
http://localhost:5000/WeatherForecast
```

## ArgoCD
## Kustomize