# Addendum and Miscellany

## ARM (Azure Resource Management) Templates

Simplest structure:

````json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "",
  "apiProfile": "",
  "parameters": {  },
  "variables": {  },
  "functions": [  ],
  "resources": [  ],
  "outputs": {  }
}
````

| Element name | Required | Description |
|:---|:---|:---|
| parameters | No | Values that are provided when deployment is executed to customize resource deployment. |
| variables | No | Values that are used as JSON fragments in the template to simplify template language expressions. |
| functions | No | User-defined functions that are available within the template. |
| resources | Yes | Resource types that are deployed or updated in a resource group or subscription. |
| outputs | No | Values that are returned after deployment. |

* limited to 256 parameters in a template. You can reduce the number of parameters by using objects that contain multiple properties.
* In the variables section, you construct values that can be used throughout your template.
* functions are limited (no variable access, only defined parameters, can't call other UDFs, can't have default values)
* In the resources section, you define the resources that are deployed or updated.
* In the Outputs section, you specify values that are returned from deployment. Typically, you return values from resources that were deployed.

Metadata

You can add a metadata object almost anywhere in your template. Resource Manager ignores the object, but your JSON editor may warn you that the property isn't valid

````json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "metadata": {
    "comments": "This template was developed for demonstration purposes.",
    "description": "User name for the Virtual Machine.",
    "author": "Example Name"
},
````

The `description` is used as a tooltip if you deploy your template in the portal

## Compute

### Containers

`az acr import`

Imports an image to an Azure Container Registry from another Container Registry. Import removes the need to docker pull, docker tag, docker push.

Import an image from 'sourceregistry' to 'MyRegistry'. The image inherits its source repository and tag names.

`az acr import -n MyRegistry --source sourceregistry.azurecr.io/sourcerepository:sourcetag`

Import an image from a public repository on Docker Hub. The image uses the specified repository and tag names.

`az acr import -n MyRegistry --source docker.io/library/hello-world:latest -t targetrepository:targettag`

Required Parameters

`--name -n`, `--source` (source image)

Optional Parameters

`--image -t` (name and tag of the image format: `-t repo/image:tag`) `--registry -r` (source Azure container registry)

#### Import container images to a container registry

Import from Docker Hub

`az acr import --name myregistry --source docker.io/library/hello-world:latest --image hello-world:latest`

Import from Microsoft Container Registry

`az acr import --name myregistry --source mcr.microsoft.com/windows/servercore:ltsc2019 --image servercore:ltsc2019`

Import from a non-Azure private container registry

`az acr import --name myregistry --source docker.io/sourcerepo/sourceimage:tag --image sourceimage:tag --username <username> --password <password>`

#### Container groups in Azure Container Instances

A container group is a collection of containers that get scheduled on the same host machine. The containers in a container group share a lifecycle, resources, local network, and storage volumes. It's similar in concept to a pod in Kubernetes.

two common ways to deploy a multi-container group: use a Resource Manager template or a YAML file.

* YAML file is recommended when your deployment includes only container instances\
* A Resource Manager template is recommended when you need to deploy additional Azure service resources

Create a deployment template

````json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "containerGroupName": {
      "type": "string",
      "defaultValue": "myContainerGroup",
      "metadata": {
        "description": "Container Group name."
      }
    }
  },
  "variables": {
    "container1name": "aci-tutorial-app",
    "container1image": "mcr.microsoft.com/azuredocs/aci-helloworld:latest",
    "container2name": "aci-tutorial-sidecar",
    "container2image": "mcr.microsoft.com/azuredocs/aci-tutorial-sidecar"
  },
  "resources": [
    {
      "name": "[parameters('containerGroupName')]",
      "type": "Microsoft.ContainerInstance/containerGroups",
      "apiVersion": "201

      ...
````

Deploy the template

`az group create --name az204-multi-container-gogo --location canadacentral`

`az deployment group create --resource-group az204-multi-container-gogo --template-file azuredeploy.json`

View deployment state

`az container show --resource-group az204-multi-container-gogo --name myContainerGroup --output table`

View container logs

`az container logs --resource-group az204-multi-container-gogo --name myContainerGroup --container-name aci-tutorial-app`

### VMs

Generalize the Windows: removes all your personal account and security information, and then prepares the machine to be used as an image.

After you have run Sysprep on a VM, that VM is considered generalized and cannot be restarted.

VM access is a system level context

User

`az identity create xxx`

System

`grant built-int role`

#### Quickstart: Create and encrypt a Linux VM with the Azure CLI

Create a resource group

`az group create --name az204-encrypt-vm --location canadacentral`

Create a virtual machine

`az vm create --resource-group az204-encrypt-vm --name az204encryptorvm --image "Canonical:UbuntuServer:16.04-LTS:latest" --size Standard_D2S_V3 --generate-ssh-keys`

Azure disk encryption stores its encryption key in an Azure Key Vault.

Create a Key Vault configured for encryption keys

`az keyvault create --name az204encryptvmkv --resource-group az204-encrypt-vm --location canadacentral --enabled-for-disk-encryption`

Encrypt the virtual machine

`az vm encryption enable --resource-group az204-encrypt-vm --name az204encryptorvm --disk-encryption-keyvault az204encryptvmkv`

`az vm encryption show --name az204encryptorvm --resource-group az204-encrypt-vm`

#### Create an image of a VM using PowerShell

````ps
 # 0. Create some variables
 $vmName = "myVM"
 $rgName = "myResourceGroup"
 $location = "EastUS"
 $imageName = "myImage"

# 1. Make sure the VM has been deallocated.
 Stop-AzVM -ResourceGroupName $rgName -Name $vmName -Force
 
# 2. Set the status of the virtual machine to Generalized.
Set-AzVm -ResourceGroupName $rgName -Name $vmName -Generalized

# 3. Get the virtual machine.
$vm = Get-AzVM -Name $vmName -ResourceGroupName $rgName

# 4. Create the image configuration.
$image = New-AzImageConfig -Location $location -SourceVirtualMachineId $vm.Id

# 5. Create the image.
New-AzImage -Image $image -ImageName $imageName -ResourceGroupName $rgName
````

Creating an image from a managed disk or a snapshot requires you to specify the managed disk ID or snapshot name respectively, but not stop or generalize the vm.

## Docker

Commands:

* FROM - set the base image. First command in the Dockerfile.
* LABEL - create metadata labels.
* ENV - set environment variables while building the Docker image.
* EXPOSE - choose what ports should be available to communicate with a container.
* RUN - run commands
* COPY - copy files from your Docker host to a Docker image
* ADD - like copy but can also extract from TARs or download over HTTP
* WORKDIR - allows you change directory while Docker builds the image, and the new directory remains the current directory for the rest of the build instructions.
* CMD - sed to execute commands as Docker launches the Docker container from the image.
* ENTRYPOINT - used as an executable for the container.

````Dockerfile
# use this image for the build stage 'base'
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1-nanoserver-1903 AS base

# Set the working directory to /app
WORKDIR /app

# documents that ports 80 and 443 (tcp) should be exposed when running this image
EXPOSE 80
EXPOSE 443

# use the core SDK for a new build stage 'build'
FROM mcr.microsoft.com/dotnet/core/sdk:3.1-nanoserver-1903 AS build

# Set the working directory to /src
WORKDIR /src

# copy everything in HOST/src/WebApplication1/WebApplication1.csproj to IMAGE/src/WebApplication11/
COPY ["WebApplication1/WebApplication1.csproj", "WebApplication1/"]

# restore any packages used by the project
RUN dotnet restore "WebApplication1/WebApplication1.csproj"

# source (now with any packages referenced) to /src
COPY . .

# Set the working directory to /src/WebApplication1
WORKDIR "/src/WebApplication1"

# Build a release version to /app/build
RUN dotnet build "WebApplication1.csproj" -c Release -o /app/build

# use build image as a new build stage 'publish'
FROM build AS publish

# publish a release version of /src/WebApplication1/WebApplication1.csproj to /app/publish
RUN dotnet publish "WebApplication1.csproj" -c Release -o /app/publish

# use the base image in build stagve 'final'
FROM base AS final

# Set /app as working directory
WORKDIR /app

# copy  published stage /app/publish to /app
COPY --from=publish /app/publish .

# set entry point to be dotnet WebApplication1.dll"
ENTRYPOINT ["dotnet", "WebApplication1.dll"]
````

### Docker Multi-stage Builds

With multi-stage builds, you use multiple FROM statements in your Dockerfile. Each FROM instruction can use a different base, and each of them begins a new stage of the build. You can selectively copy artifacts from one stage to another, leaving behind everything you don’t want in the final image.

By default, the stages are not named, and you refer to them by their integer number, starting with 0 for the first FROM instruction. However, you can name your stages, by adding an AS <NAME> to the FROM instruction

When you build your image, you don’t necessarily need to build the entire Dockerfile including every stage. You can specify a target build stage. The following command assumes you are using the previous Dockerfile but stops at the stage named builder

`$ docker build --target builder -t alexellis2/href-counter:latest .`

You can pick up where a previous stage left off by referring to it when using the FROM directive

````Dockerfile
FROM alpine:latest as builder
RUN apk --no-cache add build-base

FROM builder as build1
COPY source1.cpp source.cpp
RUN g++ -o /binary source.cpp

FROM builder as build2
COPY source2.cpp source.cpp
RUN g++ -o /binary source.cpp
````

````Dockerfile
# https://hub.docker.com/_/microsoft-dotnet-core
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build
WORKDIR /source

# copy csproj and restore as distinct layers
COPY *.sln .
COPY aspnetapp/*.csproj ./aspnetapp/
RUN dotnet restore

# copy everything else and build app
COPY aspnetapp/. ./aspnetapp/
WORKDIR /source/aspnetapp
RUN dotnet publish -c release -o /app --no-restore

# final stage/image
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1
WORKDIR /app
COPY --from=build /app ./
ENTRYPOINT ["dotnet", "aspnetapp.dll"]
````

## Storage

### Cosmos DB

#### Role-based access control in Azure Cosmos DB

| Built-in Role | Description|
|:---|:---|
| DocumentDB Account Contributor | Can manage Azure Cosmos DB accounts. |
| Cosmos DB Account Reader | Can read Azure Cosmos DB account data. |
| Cosmos Backup Operator | Can submit restore request for an Azure Cosmos database or a container. Cannot access any data or use Data Explorer. |
| Cosmos DB Operator | Can provision Azure Cosmos accounts, databases, and containers. Cannot access any data or use Data Explorer. |

RBAC support in Azure Cosmos DB applies to control plane operations only. Data plane operations are secured using primary keys or resource tokens. To learn more, see Secure access to data in Azure Cosmos DB







### Blob Storage

IF no request option provided default is PrimaryOnly for default blob client mode

#### Azure Blob storage lifecycle

GPv2 and Blob storage accounts

add, edit, or remove a policy using portal, PowerShell, cli or rest APIs

Blob service >Lifecycle Management > List View > Add a rule

Move aging data to a cooler tier

````json
{
  "rules": [
    {
      "name": "agingRule",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": [ "blockBlob" ],
          "prefixMatch": [ "container1/foo", "container2/bar" ]
        },
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 90 }
          }
        }
      }
    }
  ]
}
````

#### BlobSasBuilder Class

BlobSasBuilder is used to generate a Shared Access Signature (SAS) for an Azure Storage container or blob.

#### Grant limited access to Azure Storage resources using shared access signatures (SAS)

* User delegation SAS. A user delegation SAS is secured with Azure Active Directory (Azure AD) credentials and also by the permissions specified for the SAS. A user delegation SAS applies to Blob storage only.
* Service SAS. A service SAS is secured with the storage account key. A service SAS delegates access to a resource in only one of the Azure Storage services: Blob storage, Queue storage, Table storage, or Azure Files.
* Account SAS. An account SAS is secured with the storage account key. An account SAS delegates access to resources in one or more of the storage services. All of the operations available via a service or user delegation SAS are also available via an account SAS. 

* Ad hoc SAS: When you create an ad hoc SAS, the start time, expiry time, and permissions for the SAS are all specified in the SAS URI (or implied, if start time is omitted). Any type of SAS can be an ad hoc SAS
* Service SAS with stored access policy: A stored access policy is defined on a resource container, which can be a blob container, table, queue, or file share. 

#### Create an account SAS

````c#
static string GetAccountSASToken()
{
    // To create the account SAS, you need to use Shared Key credentials. Modify for your account.
    const string ConnectionString = "DefaultEndpointsProtocol=https;AccountName=<storage-account>;AccountKey=<account-key>";
    CloudStorageAccount storageAccount = CloudStorageAccount.Parse(ConnectionString);

    // Create a new access policy for the account.
    SharedAccessAccountPolicy policy = new SharedAccessAccountPolicy()
        {
            Permissions = SharedAccessAccountPermissions.Read | SharedAccessAccountPermissions.Write | SharedAccessAccountPermissions.List,
            Services = SharedAccessAccountServices.Blob | SharedAccessAccountServices.File,
            ResourceTypes = SharedAccessAccountResourceTypes.Service,
            SharedAccessExpiryTime = DateTime.UtcNow.AddHours(24),
            Protocols = SharedAccessProtocol.HttpsOnly
        };

    // Return the SAS token.
    return storageAccount.GetSharedAccessSignature(policy);
}
````

Use an account SAS from a client

````c#
static void UseAccountSAS(string sasToken)
{
    // Create new storage credentials using the SAS token.
    StorageCredentials accountSAS = new StorageCredentials(sasToken);
    // Use these credentials and the account name to create a Blob service client.
    CloudStorageAccount accountWithSAS = new CloudStorageAccount(accountSAS, "<storage-account>", endpointSuffix: null, useHttps: true);
    CloudBlobClient blobClientWithSAS = accountWithSAS.CreateCloudBlobClient();

    // Now set the service properties for the Blob client created with the SAS.
    blobClientWithSAS.SetServiceProperties(new ServiceProperties()
    {
        HourMetrics = new MetricsProperties()
        {
            MetricsLevel = MetricsLevel.ServiceAndApi,
            RetentionDays = 7,
            Version = "1.0"
        },
        MinuteMetrics = new MetricsProperties()
        {
            MetricsLevel = MetricsLevel.ServiceAndApi,
            RetentionDays = 7,
            Version = "1.0"
        },
        Logging = new LoggingProperties()
        {
            LoggingOperations = LoggingOperations.All,
            RetentionDays = 14,
            Version = "1.0"
        }
    });

    // The permissions granted by the account SAS also permit you to retrieve service properties.
    ServiceProperties serviceProperties = blobClientWithSAS.GetServiceProperties();
    Console.WriteLine(serviceProperties.HourMetrics.MetricsLevel);
    Console.WriteLine(serviceProperties.HourMetrics.RetentionDays);
    Console.WriteLine(serviceProperties.HourMetrics.Version);
}
````

## Security

### Azure AD

#### Secure user sign-in events with Azure Multi-Factor Authentication

Create a Conditional Access policy to enable Azure Multi-Factor Authentication for a group of users

Azure Active Directory > Security > Conditional Access > New policy

name: MFA Pilot

Assignments > Users and groups > Select users and groups

Configure the conditions for multi-factor authentication

Cloud apps or actions > Select apps > Microsoft Azure Management > Select > Done

Set the Enable policy toggle to On.

To apply the Conditional Access policy, select Create.

Test Azure Multi-Factor Authentication

Log into the portal as a non-admin. Required to register and use 2FA.

#### Add group claims to tokens

select which groups should be included in the token (ex: Sto emit all the Security Groups the user is a member of, select Security Groups)

#### Configure TLS mutual authentication for Azure App Service

You can restrict access to your Azure App Service app by enabling different types of authentication for it. One way to do it is to request a client certificate when the client request is over TLS/SSL and validate the certificate. This mechanism is called TLS mutual authentication or client certificate authentication. This article shows how to set up your app to use client certificate authentication.

Enable client certificates

`az webapp update --set clientCertEnabled=true --name <app_name> --resource-group <group_name>`

Exclude paths from requiring authentication

Exclusion paths can be configured by selecting Configuration > General Settings and defining an exclusion path (example /public)

Access client certificate

In App Service, TLS termination of the request happens at the frontend load balancer. When forwarding the request to your app code with client certificates enabled, App Service injects an `X-ARR-ClientCert` request header with the client certificate. App Service does not do anything with this client certificate other than forwarding it to your app. Your app code is responsible for validating the client certificate.

For ASP.NET, the client certificate is available through the `HttpRequest.ClientCertificate` property.

## Optimization

### Content Delivery Networks

#### Redis

Redis Commands:

* PING: used to test if a connection is still alive, or to measure latency.
* INFO:  returns information and statistics about the server
* SET: Set key to hold the string value. If key already holds a value, it is overwritten, regardless of its type.
* SETEX: Set key to hold the string value and set key to timeout after a given number of seconds. Same as:

````redis
SET mykey value
EXPIRE mykey seconds
````

* GET: Get the value of key. If the key does not exist the special value nil is returned. An error is returned if the value stored at key is not a string, because GET only handles string value

## Azure Services

### Azure Event Hubs

#### Overview

ingests and processes large volumes of events and data

unique scoping container to a namespace

Any entity that sends data to an event hub is an event producer

Using publisher policies, each publisher uses its own unique identifier when publishing events to an event hub

`//<my namespace>.servicebus.windows.net/<event hub name>/publishers/<my publisher name>`

Any entity that reads event data from an event hub is an event consumer

The publish/subscribe mechanism of Event Hubs is enabled through consumer groups.

The number of partitions is specified at creation and must be between 2 and 32. The partition count is not changeable, so you should consider long-term scale when setting partition count. Partitions are a data organization mechanism that relates to the downstream parallelism required in consuming applications. The number of partitions in an event hub directly relates to the number of concurrent readers you expect to have.

Checkpointing is a process by which readers mark or commit their position within a partition event sequence.

#### capturing of events streaming

Azure Event Hubs Capture enables you to automatically deliver the streaming data in Event Hubs to an Azure Blob storage or Azure Data Lake Storage Gen1 or Gen 2 account of your choice.

Capture: ON

Capture Provider: Storage or Data Lake

Azure Storage Container: set container

### Service Bus

#### Topic filters and actions

Subscribers can define which messages they want to receive from a topic.

Three filter conditions:

1. B`oolean filters` - The `TrueFilter` and `FalseFilter` either cause all arriving messages (true) or none of the arriving messages (false) to be selected for the subscription.
2. `SQL Filters` - A SqlFilter holds a SQL-like conditional expression that is evaluated in the broker against the arriving messages' user-defined properties and system properties.
3. `Correlation Filters` - A CorrelationFilter holds a set of conditions that are matched against one or more of an arriving message's user and system properties.
    * A match exists when an arriving message's value for a property is equal to the value specified in the correlation filter. For string expressions, the comparison is case-sensitive. When specifying multiple match properties, the filter combines them as a logical AND condition, meaning for the filter to match, all conditions must match.

### Logic Apps

using templates

````json
	Resources:
        type: Microsoft.Logic/workflows
````

an integration account, which is a separate Azure resource that provides a secure, scalable, and manageable container for the integration artifacts that you define and use with your logic app workflows.

