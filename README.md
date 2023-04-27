# Gettomg Started with Kubernetes

## Prerequisites 
* .NET 7 SDK & Runtime
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
        image: jeanfrg/app1:
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

Argo CD is an open-source, Kubernetes-native Continuous Delivery (CD) tool that automates the deployment, scaling, and management of applications in Kubernetes clusters. It follows the GitOps methodology, which uses Git as the single source of truth for declarative infrastructure and application configurations.

### Setting up ArgoCD

To install ArgoCD in Kubernetes we first need to create the ArgoCD namespace. Kubernetes namespaces are a way to divide cluster resources among multiple users, teams, or projects.

```bash
kubectl create namespace argocd
```

Next we need to apply the ArgoCD manifests

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/master/manifests/install.yaml
```

Sources

[https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd](https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd)
[https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#non-high-availability](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#non-high-availability)

To expose ArgoCD we need to change the directory to `kubernetes/apps/argocd` and patch its `argocd-server` service.

```bash
kubectl patch svc argocd-server -n argocd --type=merge --patch-file argocd-server.service.patch.yaml
```

With this patch we're now able to access the service outside of Kubernetes using `http://localhost:5001/`. The initial password for the admin user account can be found inside the secret `argocd-initial-admin-secret`.

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}'
```

The value is bse64 encoded, it can be decoded in various ways, however it depends on the shell you're using.

Ubuntu/WSL
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 --decode
```

Windows/PowerShell
```powershell
([System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}'))))
```

Using the username `admin` and the password retreived from wither commands, we're now able to login to ArgoCD.

### Configuring the application for ArgoCD

We're now ready to configure ArgoCD to manage the deployment of our application when our Kubernetes manifests for our application change.

To achieve this, we need to create a manifest of kind Application in the directory `kubernetes/apps/argocd/applications` with the name `app1.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app1
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/frg/k8-containerization-demo.git'
    targetRevision: main
    path: kubernetes/apps/app1
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Change the directory to `kubernetes/apps/argocd/applications` and apply the manifest in kubernetes.

```bash
kubectl apply -f app1.yaml
```

Checking the ArgoCD UI we should now be able to see our application and its sync status. To test if ArgoCD can successfully pick up manifest changes, we can open the `deployment.yaml` in `kubernetes/apps/app1`, change `spec.replicas` to `2`, commit and push our changes.

By default ArgoCD polls the configured repository specified in `kubernetes/apps/argocd/applications/app1.yaml` at `spec.source.repoURL` every 180 seconds.

Once the changes have been picked up we will be able to see 2 pods that have been deployed instead of 1. For this demo we don't need two so feel free to reduce it back to 1.

## Kustomize

Kustomize is a tool for customizing Kubernetes resources through a base and overlay approach. It is designed to make it easy to manage and maintain Kubernetes resource configurations across different environments (e.g., development, staging, production) without requiring manual changes or complex templating systems.

The key components of Kustomize are:

1. **Base:** The base resources that define the core structure and functionality of an application.
2. **Overlay:** A set of modifications applied to the base resources to adapt them for a specific environment or purpose.
3. **kustomization.yaml:** A configuration file that defines how to combine the base resources and overlays, including instructions for merging and patching resources.

