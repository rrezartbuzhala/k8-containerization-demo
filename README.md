# Gettomg Started with Kubernetes

## Prerequisites 
* .NET SDK installed
* Docker Desktop

## Scenario
You have two applications and they need to be setup in a way where they're able to be deployed into a Kubernetes cluster.

The applications can be found in the `apps` folder, they are both identicle and where generated using the dotnet template command.

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

Currently neither of our applications are containerized. To containerize them we first need to create a `Dockerfile` in the root of each project, and paste the following into the files.

```bash
# Use the official Microsoft .NET runtime image as the base image
FROM mcr.microsoft.com/dotnet/aspnet:5.0

# Set the working directory inside the container
WORKDIR /app

# Copy the published output of the application to the container
COPY ./publish /app

# Expose the port the application will run on
EXPOSE 80

# Start the application using the dotnet runtime
ENTRYPOINT ["dotnet", "app1.dll"]
```

Note that in the `ENTRYPOINT` command there is a reference to `app1` in this example. This should be changed to the project name of your application.

In the `COPY` command there is a reference to a local directory called `publish`. This directory does not exist yet, however we can fix that by publishing our application.

```bash
dotnet publish --configuration Release --output ./publish
```



## Kubernetes
## ArgoCD
## Kustomize