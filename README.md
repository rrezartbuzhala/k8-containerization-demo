# Gettomg Started with Kubernetes

## Prerequisites 
* .NET 7 SDK & Runtime
* Docker Desktop

## Scenario
You have an application and it needs to be setup in a way where it's able to be deployed to Kubernetes. The application can be found in the `apps` folder, it was generated using the dotnet template command.

```bash
# apps/
dotnet new webapi -o app1
```

This command uses a native dotnet template `webapi` to create a web API project that generates fake weather data.

To run the application change directory `cd apps/app1`, then execute `dotnet run` to build and launch the application. Using the port noted in the console logs you can append `/swagger`, and the Swagger documentation will be loaded.

## Containerization
Containerization is the act of creating portable, isolated, and reproducible environments for applications. Docker is the most popular tool to do this, however there's also other tools like Containerd. For the purpose of this demo we will use Docker throughout.

Within the context of Docker there is a question that we first need to answer. What is the difference between a `Dockerfile` and a `docker-compose` file?

* **Dockerfile**: used to build a Docker Image. a Docker Image contains everything related to the environment that an application needs.
* **docker-compose**: used to manage one or more images including setting up storage volumes, virtual networks, container resources, and other images that might be a dependency for an application like a database. These qualities make it ideal for local environments.

### Containerizing our applications

To containerize our application we first need to create a `Dockerfile` in the root of the project.

```bash
# apps/app1/
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

Note that in the `ENTRYPOINT` command there is a reference to `app1` -- this should be changed to the project name of your application.

In the `COPY` command there is a reference to a local directory called `publish`. This directory does not exist yet, however we can fix that by publishing our application.

```bash
# apps/app1/
dotnet publish --configuration Release --output ./publish
```

Next we generate our Docker image. The Docker image will be registered in Docker Desktop's local image registry.

```bash
# apps/app1/
docker build --tag app1 .
```

To test our new Docker image we can start a container using the image we just created.

```bash
# apps/app1/
docker run --rm -p 5000:8080 --name my_app1 app1
```

Loading the url `http://localhost:5000/WeatherForecast` in a browser or executing a cURL should return a JSON response with a list of fake weather forecasts.

```bash
curl -XGET 'http://localhost:5000/WeatherForecast'
```

Note that the Swagger endpoint we used earlier is not enabled on production environments by default. In order to change the environment of your application you can set the environment variable `ASPNETCORE_ENVIRONMENT` to `Development` in the Dockerfile, or modify the code within `Program.cs`.

If an error comes up that the container is already running it should be stopped. Since we're using the `--rm` command to start the container, the container will be removed automatically.

```bash
docker stop my_app1
```

The final step is to publish our image to a remote Docker registery. The app used for this Demo is already available as the image `jeanfrg/app1`, however to publish your own we need to use these commands.

```bash
docker tag app1 <your-docker-hub-id>/app1:v1
docker push <your-docker-hub-id>/app1:v1
```

## Kubernetes

Kubernetes is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications -- we call these tools Orchestrators.

### Setting up the local environment

There are multiple ways that one can set up a local kubernetes environment. Since we're using Docker already the easiest way is to use Docker's native integration.

[Enable and start Kubernetes](https://docs.docker.com/desktop/kubernetes/#enable-kubernetes) on Docker Desktop.

[MiniKube](https://minikube.sigs.k8s.io/docs/start/) is also a good tool that we can use to achieve this.

Test if the installation was successful by getting the Kubernetes version.

```bash
kubectl version --short
```

### Setting up our application

Next we need to create some Kubernetes manifests, which tell Kubernetes how to configure our application.

```yaml
# kubernetes/apps/app1/deployment.yaml
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
        image: jeanfrg/app1:v1.0.10
        ports:
        - containerPort: 8080
```

This configuration file defines a deployment with one replica of our Docker image, and exposes port 8080 on the container.

Note that if you have published your own docker image, `spec.template.spec.[].image` should reference your image.

Next we need to create a Kubernetes service which will allow us to expose the service out of Kubernetes so we can hit it locally.

```yaml
# kubernetes/apps/app1/service.yaml
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

This manifest file defines a LoadBalancer service that exposes your application on port 5000 and routes traffic to the target port 8080 on the deployment.

Apply configurations.

```bash
# kubernetes/apps/app1/
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Note we can also change the directory to `kubernetes/apps` and use `kubectl apply -f app1` to apply all configurations within the folder without specifying each one.

Checking the configuration status in Kubernetes.

```bash
kubectl get deployments
kubectl get services
```

The information printed out from the services command will show us exactly how we're able to access our application from outside Kubernetes.

```
# kubectl get services
NAME         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
app1         LoadBalancer   10.101.183.235   localhost     5000:31785/TCP   1m
```

In this example we use the value of the external IP column as the host and the first port specified in the port column.

```
http://localhost:5000/WeatherForecast
```

Loading this URL should return a JSON response with fake weather forcasts.

## ArgoCD

Argo CD is an open-source, Kubernetes-native Continuous Delivery (CD) tool that automates the deployment, scaling, and management of applications in Kubernetes clusters. It follows the GitOps methodology, which uses Git as the single source of truth for declarative infrastructure and application configurations.

### Setting up ArgoCD

To install ArgoCD in Kubernetes we first need to create the ArgoCD namespace. Kubernetes namespaces are a way to divide cluster resources among multiple users, teams, or projects.

```bash
kubectl create namespace argocd
```

Next we need to apply the ArgoCD manifests.

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
```

Sources

[https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd](https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd)
[https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#non-high-availability](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#non-high-availability)

To expose ArgoCD we need to patch its `argocd-server` service.

```bash
# kubernetes/apps/argocd/
kubectl patch svc argocd-server -n argocd --type=merge --patch-file argocd-server.service.patch.yaml
```

With this patch we're now able to access the service outside of Kubernetes using `http://localhost:5010/`. The initial password for the admin user account can be found inside the secret `argocd-initial-admin-secret`.

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

Using the username `admin` and the password retreived from either commands, we're now able to login to ArgoCD.

### Configuring the application for ArgoCD

We're now ready to configure ArgoCD to manage the deployment of our application when our Kubernetes manifests for our application change.

To achieve this, we need to create a manifest of kind Application.

```yaml
# kubernetes/apps/app1/application.yaml
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

At this point we're ready to apply these changes. ArgoCD will attempt to pick up changes from the specified repository in the `application.yaml`. This means it's time to commit and push our code to that repository.

```bash
# kubernetes/apps/app1/
kubectl apply -f application.yaml
```

Checking the ArgoCD UI we should now be able to see our application and its sync status. To test if ArgoCD can successfully pick up manifest changes, we can open the `deployment.yaml`, change `spec.replicas` to `2`, commit and push our changes.

By default ArgoCD polls the configured repository specified in `kubernetes/apps/app1/application.yaml` at `spec.source.repoURL` every 180 seconds.

Once the changes have been picked up we will be able to see 2 pods that have been deployed instead of 1. For this demo we don't need two so feel free to reduce it back to 1.

## Kustomize

Kustomize is a tool for customizing Kubernetes resources through a base and overlay approach. It is designed to make it easy to manage and maintain Kubernetes resource configurations across different environments (e.g., development, staging, production) without requiring manual changes or complex templating systems.

The key components of Kustomize are:

1. **Base:** The base resources that define the core structure and functionality of an application.
2. **Overlay:** A set of modifications applied to the base resources to adapt them for a specific environment or purpose.
3. **kustomization.yaml:** A configuration file that defines how to combine the base resources and overlays, including instructions for merging and patching resources.

### Restructing our application for multiple environments

Let's say we need this application to be deployed for the development and production environments. We can either take the approach of copying each Kubernetes manifest for every environment or we can use Kustomize to reuse manifests per environment.

First we need to create these directories.

```
kubernetes/apps/app1/bases
kubernetes/apps/app1/overlays/development
kubernetes/apps/app1/overlays/production
```

Move the manifests found at `kubernetes/apps/app1/` to `kubernetes/apps/app1/bases`. As implied, we will use thes manifests as the base of each environment manifest.

Create a kustomization file that will be referenced within each overlay.

```yaml
# kubernetes/apps/app1/bases/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

We need to create a kustomization file for development.

```yaml
# kubernetes/apps/app1/overlays/development/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namePrefix: dev-

resources:
  - ../../base
```

Two differences that we want the production deployment to have is a different port and an increased amount of replicas. To achieve this we use the concept of patches, which are used to modify resources.

```yaml
# kubernetes/apps/app1/overlays/production/deployment.patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 3
```

```yaml
# kubernetes/apps/app1/overlays/production/service.patch.yaml
- op: replace
  path: /spec/ports/0/port
  value: 5001
```

Next, we need to create the production `kustomization.yaml` so that it references these patch files.

```yaml
# kubernetes/apps/app1/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namePrefix: prod-

resources:
  - ../../base

patches:
  - path: deployment.patch.yaml
  - target:
      version: v1
      kind: Service
      name: app1
    path: service.patch.yaml
```

The ArgoCD application manifest also need to be created for both these deployments.

```yaml
# kubernetes/apps/app1/overlays/development/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-app1
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/frg/k8-containerization-demo.git'
    targetRevision: main
    path: kubernetes/apps/app1/overlays/development
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```yaml
# kubernetes/apps/app1/overlays/production/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prod-app1
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/frg/k8-containerization-demo.git'
    targetRevision: main
    path: kubernetes/apps/app1/overlays/production
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Commit these changes to your repository and finally, we can apply these manifests.

```bash
# kubernetes/apps/app1/overlays/production
kubectl apply -f application.yaml
```

```bash
# kubernetes/apps/app1/overlays/development
kubectl apply -f application.yaml
```