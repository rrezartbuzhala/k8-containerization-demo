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