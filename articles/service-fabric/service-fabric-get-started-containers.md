﻿---
title: Create an Azure Service Fabric container application | Microsoft Docs
description: Create your first Windows container application on Azure Service Fabric.  Build a Docker image with a Python application, push the image to a container registry, build and deploy a Service Fabric container application.
services: service-fabric
documentationcenter: .net
author: rwike77
manager: timlt
editor: 'vturecek'

ms.assetid: 
ms.service: service-fabric
ms.devlang: dotNet
ms.topic: get-started-article
ms.tgt_pltfrm: NA
ms.workload: NA
ms.date: 06/30/2017
ms.author: ryanwi

---

# Create your first Service Fabric container application on Windows
> [!div class="op_single_selector"]
> * [Windows](service-fabric-get-started-containers.md)
> * [Linux](service-fabric-get-started-containers-linux.md)

Running an existing application in a Windows container on a Service Fabric cluster doesn't require any changes to your application. This article walks you through creating a Docker image containing a Python [Flask](http://flask.pocoo.org/) web application and deploying it to a Service Fabric cluster.  You will also share your containerized application through [Azure Container Registry](/azure/container-registry/).  This article assumes a basic understanding of Docker. You can learn about Docker by reading the [Docker Overview](https://docs.docker.com/engine/understanding-docker/).

## Prerequisites
A development computer running:
* Visual Studio 2015 or Visual Studio 2017.
* [Service Fabric SDK and tools](service-fabric-get-started.md).
*  Docker for Windows.  [Get Docker CE for Windows (stable)](https://store.docker.com/editions/community/docker-ce-desktop-windows?tab=description). After installing and starting Docker, right-click on the tray icon and select **Switch to Windows containers**. This is required to run Docker images based on Windows.

A Windows cluster with three or more nodes running on Windows Server 2016 with Containers- [Create a cluster](service-fabric-cluster-creation-via-portal.md) or [try Service Fabric for free](https://aka.ms/tryservicefabric). 

A registry in Azure Container Registry - [Create a container registry](../container-registry/container-registry-get-started-portal.md) in your Azure subscription. 

## Define the Docker container
Build an image based on the [Python image](https://hub.docker.com/_/python/) located on Docker Hub. 

Define your Docker container in a Dockerfile. The Dockerfile contains instructions for setting up the environment inside your container, loading the application you want to run, and mapping ports. The Dockerfile is the input to the `docker build` command, which creates the image. 

Create an empty directory and create the file *Dockerfile* (with no file extension). Add the following to *Dockerfile* and save your changes:

```
# Use an official Python runtime as a base image
FROM python:2.7-windowsservercore

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
ADD . /app

# Install any needed packages specified in requirements.txt
RUN pip install -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```

Read the [Dockerfile reference](https://docs.docker.com/engine/reference/builder/) for more information.

## Create a simple web application
Create a Flask web application listening on port 80 that returns "Hello World!".  In the same directory, create the file *requirements.txt*.  Add the following and save your changes:
```
Flask
```

Also create the *app.py* file and add the following:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    
    return 'Hello World!'

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```

## Build the image
Run the `docker build` command to create the image that runs your web application. Open a PowerShell window and navigate to the directory containing the Dockerfile. Run the following command:

```
docker build -t helloworldapp .
```

This command builds the new image using the instructions in your Dockerfile, naming (-t tagging) the image "helloworldapp". Building an image pulls the base image down from Docker Hub and creates a new image that adds your application on top of the base image.  

Once the build command completes, run the `docker images` command to see information on the new image:

```
$ docker images
    
REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
helloworldapp                 latest              8ce25f5d6a79        2 minutes ago       10.4 GB
```

## Run the application locally
Verify your image locally before pushing it the container registry.  

Run the application:

```
docker run -d --name my-web-site helloworldapp
```

*name* gives a name to the running container (instead of the container ID).

Once the container starts, find its IP address so that you can connect to your running container from a browser:
```
docker inspect -f "{{ .NetworkSettings.Networks.nat.IPAddress }}" my-web-site
```

Connect to the running container.  Open a web browser pointing to the IP address returned, for example "http://172.31.194.61". You should see the heading "Hello World!" display in the browser.

To stop your container, run:

```
docker stop my-web-site
```

Delete the container from your development machine:

```
docker rm my-web-site
```

## Push the image to the container registry
After you verify that the container runs on your development machine, push the image to your registry in Azure Container Registry.

Run ``docker login`` to log in to your container registry with your [registry credentials](../container-registry/container-registry-authentication.md).

The following example passes the ID and password of an Azure Active Directory [service principal](../active-directory/active-directory-application-objects.md). For example, you might have assigned a service principal to your registry for an automation scenario. Or, you could login using your registry username and password.

```
docker login myregistry.azurecr.io -u xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx -p myPassword
```

The following command creates a tag, or alias, of the image, with a fully qualified path to your registry. This example places the image in the ```samples``` namespace to avoid clutter in the root of the registry.

```
docker tag helloworldapp myregistry.azurecr.io/samples/helloworldapp
```

Push the image to your container registry:

```
docker push myregistry.azurecr.io/samples/helloworldapp
```

## Create and package the containerized service in Visual Studio
The Service Fabric SDK and tools provide a service template to help you deploy a container to a Service Fabric cluster.

1. Start Visual Studio.  Select **File** > **New** > **Project**.
2. Select **Service Fabric application**, name it "MyFirstContainer", and click **OK**.
3. Select **Guest Container** from the list of **service templates**.
4. In **Image Name** enter "myregistry.azurecr.io/samples/helloworldapp", the image you pushed to your container repository. 
5. Give your service a name, and click **OK**.
6. If your containerized service needs an endpoint for communication, you can now add the protocol, port, and type to an ```Endpoint``` in the ServiceManifest.xml file. For this article, the containerized service listens on port 80: 

    ```xml
    <Endpoint Name="Guest1TypeEndpoint" UriScheme="http" Port="8081" Protocol="http"/>
    ```
    Providing the ```UriScheme``` automatically registers the container endpoint with the Service Fabric Naming service for discoverability. A full ServiceManifest.xml example file is provided at the end of this article. 
7. Configure the container port-to-host port mapping by specifying a ```PortBinding``` policy in ```ContainerHostPolicies``` of the ApplicationManifest.xml file.  For this article, ```ContainerPort``` is 8081 (the container exposes port 80, as specified in the Dockerfile) and ```EndpointRef``` is "Guest1TypeEndpoint" (the endpoint defined in the service manifest).  Incoming requests to the service on port 8081 are mapped to port 80 on the container.  If your container needs to authenticate with a private repository, then add ```RepositoryCredentials```.  For this article, add the account name and password for the myregistry.azurecr.io container registry. 

    ```xml
    <Policies>
        <ContainerHostPolicies CodePackageRef="Code">
            <RepositoryCredentials AccountName="myregistry" Password="=P==/==/=8=/=+u4lyOB=+=nWzEeRfF=" PasswordEncrypted="false"/>
            <PortBinding ContainerPort="80" EndpointRef="Guest1TypeEndpoint"/>
        </ContainerHostPolicies>
    </Policies>
    ```

    A full ApplicationManifest.xml example file is provided at the end of this article.
8. Configure the cluster connection endpoint so you can publish the application to your cluster.  You can find the client connection endpoint in the Overview blade for your cluster in the [Azure portal](https://portal.azure.com). In Solution Explorer, open *Cloud.xml* under **MyFirstContainer**->**PublishProfiles**.  Add the cluster name and connection port to **ClusterConnectionParameters**.  For example:
    ```xml
    <ClusterConnectionParameters ConnectionEndpoint="containercluster.westus2.cloudapp.azure.com:19000" />
    ```
    
9. Save all files and build your project.  

10. To package your application, right-click on **MyFirstContainer** in Solution Explorer and select **Package**. 

## Deploy the container application
To publish your application, right-click on **MyFirstContainer** in Solution Explorer and select **Publish**.

[Service Fabric Explorer](service-fabric-visualizing-your-cluster.md) is a web-based tool for inspecting and managing applications and nodes in a Service Fabric cluster. Open a browser and navigate to http://containercluster.westus2.cloudapp.azure.com:19080/Explorer/ and follow the application deployment.  The application deploys but is in an error state until the image is downloaded on the cluster nodes (which can take some time, depending on the image size):
![Error][1]

The application is ready when it's in ```Ready``` state:
![Ready][2]

Open a browser and navigate to http://containercluster.westus2.cloudapp.azure.com:8081. You should see the heading "Hello World!" display in the browser.

## Clean up
You continue to incur charges while the cluster is running, consider [deleting your cluster](service-fabric-get-started-azure-cluster.md#remove-the-cluster).  [Party clusters](http://tryazureservicefabric.westus.cloudapp.azure.com/) are automatically deleted after a few hours.

After you push the image to the container registry you can delete the local image from your development computer:

```
docker rmi helloworldapp
docker rmi myregistry.azurecr.io/samples/helloworldapp
```

## Complete example Service Fabric application and service manifests
Here are the complete service and application manifests used in this article.

### ServiceManifest.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<ServiceManifest Name="Guest1Pkg"
                 Version="1.0.0"
                 xmlns="http://schemas.microsoft.com/2011/01/fabric"
                 xmlns:xsd="http://www.w3.org/2001/XMLSchema"
                 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <ServiceTypes>
    <!-- This is the name of your ServiceType.
         The UseImplicitHost attribute indicates this is a guest service. -->
    <StatelessServiceType ServiceTypeName="Guest1Type" UseImplicitHost="true" />
  </ServiceTypes>

  <!-- Code package is your service executable. -->
  <CodePackage Name="Code" Version="1.0.0">
    <EntryPoint>
      <!-- Follow this link for more information about deploying Windows containers to Service Fabric: https://aka.ms/sfguestcontainers -->
      <ContainerHost>
        <ImageName>myregistry.azurecr.io/samples/helloworldapp</ImageName>
      </ContainerHost>
    </EntryPoint>
    <!-- Pass environment variables to your container: -->
    <!--
    <EnvironmentVariables>
      <EnvironmentVariable Name="VariableName" Value="VariableValue"/>
    </EnvironmentVariables>
    -->
  </CodePackage>

  <!-- Config package is the contents of the Config directoy under PackageRoot that contains an 
       independently-updateable and versioned set of custom configuration settings for your service. -->
  <ConfigPackage Name="Config" Version="1.0.0" />

  <Resources>
    <Endpoints>
      <!-- This endpoint is used by the communication listener to obtain the port on which to 
           listen. Please note that if your service is partitioned, this port is shared with 
           replicas of different partitions that are placed in your code. -->
      <Endpoint Name="Guest1TypeEndpoint" UriScheme="http" Port="8081" Protocol="http"/>
    </Endpoints>
  </Resources>
</ServiceManifest>
```
### ApplicationManifest.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<ApplicationManifest ApplicationTypeName="MyFirstContainerType"
                     ApplicationTypeVersion="1.0.0"
                     xmlns="http://schemas.microsoft.com/2011/01/fabric"
                     xmlns:xsd="http://www.w3.org/2001/XMLSchema"
                     xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <Parameters>
    <Parameter Name="Guest1_InstanceCount" DefaultValue="-1" />
  </Parameters>
  <!-- Import the ServiceManifest from the ServicePackage. The ServiceManifestName and ServiceManifestVersion 
       should match the Name and Version attributes of the ServiceManifest element defined in the 
       ServiceManifest.xml file. -->
  <ServiceManifestImport>
    <ServiceManifestRef ServiceManifestName="Guest1Pkg" ServiceManifestVersion="1.0.0" />
    <ConfigOverrides />
    <Policies>
      <ContainerHostPolicies CodePackageRef="Code">
        <RepositoryCredentials AccountName="myregistry" Password="=P==/==/=8=/=+u4lyOB=+=nWzEeRfF=" PasswordEncrypted="false"/>
        <PortBinding ContainerPort="80" EndpointRef="Guest1TypeEndpoint"/>
      </ContainerHostPolicies>
    </Policies>
  </ServiceManifestImport>
  <DefaultServices>
    <!-- The section below creates instances of service types, when an instance of this 
         application type is created. You can also create one or more instances of service type using the 
         ServiceFabric PowerShell module.
         
         The attribute ServiceTypeName below must match the name defined in the imported ServiceManifest.xml file. -->
    <Service Name="Guest1">
      <StatelessService ServiceTypeName="Guest1Type" InstanceCount="[Guest1_InstanceCount]">
        <SingletonPartition />
      </StatelessService>
    </Service>
  </DefaultServices>
</ApplicationManifest>
```

## Next steps
* Learn more about running [containers on Service Fabric](service-fabric-containers-overview.md).
* Read the [Deploy a .NET application in a container](service-fabric-host-app-in-a-container.md) tutorial.
* Learn about the Service Fabric [application life-cycle](service-fabric-application-lifecycle.md).
* Checkout the [Service Fabric container code samples](https://github.com/Azure-Samples/service-fabric-dotnet-containers) on GitHub.

[1]: ./media/service-fabric-get-started-containers/MyFirstContainerError.png
[2]: ./media/service-fabric-get-started-containers/MyFirstContainerReady.png
