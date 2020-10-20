# Develop Azure Compute Solutions (25-30%)

![Candidate Compute Services](img/azure-compute-decision-tree.svg)

## Implement Infrastructure-as-a-Service (IaaS) solutions

* provision individual VMs along with the associated networking and storage components
* Deploy whatever software and applications you want onto those VMs
* You still manage the individual VMs inside an Azure virtual network

Flexibility of virtualization without buying and maintaining the physical hardware.

#### Components of a Provisioned VM

* Virtual machine
* Public IP address
* Network security group
* Network interface
* Disk
* Virtual network

### provision VMs

Considerations

* names of application resources
* location where resources will be stored
* VM sizing and capacity
* Max VMs that can be created
* OS
* configuration
* related resources

### Portal

Services > Virtual Machines  > Add

Basics

* Subscription
* Resource Group

Instance Details

* VM Name
* Region
* Availability
* Image
* Size

Define the Administrator Account

Set any Inbound port rules (like RDP 3389 and HTTP 80)

Review > Create

#### Connect via RDP

Connect > RDP > Download & Open

Use defined administrator credentials (local account)

Install IIS via PowerShell

`Install-WindowsFeature -name Web-Server -IncludeManagementTools`

close the RDP Connection

Visit the IP in a web browser

### CLI

Create a resource group

`az group create --name myResourceGroup --location eastus`

Create virtual machine

````
az vm create \
    --resource-group myResourceGroup \
    --name myVM \
    --image win2016datacenter \
    --admin-username azureuser
````

Note your own `publicIpAddress` in the output from your VM.

Open port 80 for web traffic

`az vm open-port --port 80 --resource-group myResourceGroup --name myVM`

Connect to virtual machine

`mstsc /v:publicIpAddress`

Install web server (via PowerShell)

`Install-WindowsFeature -name Web-Server -IncludeManagementTools`

### PowerShell

Create a resource group

`New-AzResourceGroup -Name myResourceGroup -Location EastUS`

Create virtual machine

````PowerShell
New-AzVm `
    -ResourceGroupName "myResourceGroup" `
    -Name "myVM" `
    -Location "East US" `
    -VirtualNetworkName "myVnet" `
    -SubnetName "mySubnet" `
    -SecurityGroupName "myNetworkSecurityGroup" `
    -PublicIpAddressName "myPublicIpAddress" `
    -OpenPorts 80,3389
````

Get your public IP address

`Get-AzPublicIpAddress -ResourceGroupName "myResourceGroup" | Select "IpAddress"`

Connect to virtual machine

`mstsc /v:publicIpAddress`

Install web server

`Install-WindowsFeature -name Web-Server -IncludeManagementTools`

### ARM template

https://docs.microsoft.com/en-us/azure/virtual-machines/windows/quick-create-template

Resources defined in the Create VM template:

* Microsoft.Network/virtualNetworks/subnets: create a subnet.
* Microsoft.Storage/storageAccounts: create a storage account.
* Microsoft.Network/publicIPAddresses: create a public IP address.
* Microsoft.Network/networkSecurityGroups: create a network security group.
* Microsoft.Network/virtualNetworks: create a virtual network.
* Microsoft.Network/networkInterfaces: create a NIC.
* Microsoft.Compute/virtualMachines: create a virtual machine.

### Configure VMs for Remote Access 

Lock down inbound traffic to your Azure Virtual Machines with Azure Security Center's just-in-time (JIT) virtual machine (VM) access feature.

* Enable JIT on your VMs - You can enable JIT with your own custom options for one or more VMs using Security Center, PowerShell, or the REST API. Alternatively, you can enable JIT with default, hard-coded parameters, from Azure virtual machines. When enabled, JIT locks down inbound traffic to your Azure VMs by creating a rule in your network security group.
* Request access to a VM that has JIT enabled - The goal of JIT is to ensure that even though your inbound traffic is locked down, Security Center still provides easy access to connect to VMs when needed. You can request access to a JIT-enabled VM from Security Center, Azure virtual machines, PowerShell, or the REST API.
* Audit the activity - To ensure your VMs are secured appropriately, review the accesses to your JIT-enabled VMs as part of your regular security checks.

#### Enable JIT VM access

Security Center > Just in time VM access > Add

Virtual Machines > VM Instance > Configuration > Just-in-time access > enable

PowerShell

Enable just-in-time VM access on a specific VM with the following rules:

* Close ports 22 and 3389
* Set a maximum time window of 3 hours for each so they can be opened per approved request
* Allow the user who is requesting access to control the source IP addresses
* Allow the user who is requesting access to establish a successful session upon an approved just-in-time access request

Assign a variable that holds the just-in-time VM access rules for a VM:

````PowerShell
$JitPolicy = (@{
    id="/subscriptions/SUBSCRIPTIONID/resourceGroups/RESOURCEGROUP/providers/Microsoft.Compute/virtualMachines/VMNAME";
    ports=(@{
         number=22;
         protocol="\*";
         allowedSourceAddressPrefix=@("\*");
         maxRequestAccessDuration="PT3H"},
         @{
         number=3389;
         protocol="\*";
         allowedSourceAddressPrefix=@("\*");
         maxRequestAccessDuration="PT3H"})})
````

Insert the VM just-in-time VM access rules into an array:

`$JitPolicyArr=@($JitPolicy)`

Configure the just-in-time VM access rules on the selected VM:

`Set-AzJitNetworkAccessPolicy -Kind "Basic" -Location "LOCATION" -Name "default" -ResourceGroupName "RESOURCEGROUP" -VirtualMachine $JitPolicyArr`

#### Request access to a JIT-enabled VM

Portal

Virtual Machines > VM Instance > Connect > Request access

PowerShell

Define the request access properties, insert the VM access request parameters in an array,

`Start-AzJitNetworkAccessPolicy -ResourceId "/subscriptions/SUBSCRIPTIONID/resourceGroups/RESOURCEGROUP/providers/Microsoft.Security/locations/LOCATION/jitNetworkAccessPolicies/default" -VirtualMachine $JitPolicyArr`

#### Audit JIT access activity in Security Center

Security Center > Just in time VM access > Configured . Activity Log

### Create ARM Templates

Templates sections:

* Parameters - Provide values during deployment that allow the same template to be used with different environments.
* Variables - Define values that are reused in your templates. They can be constructed from parameter values.
* User-defined functions - Create customized functions that simplify your template.
* Resources - Specify the resources to deploy.
* Outputs - Return values from the deployed resources.

Portal

* New: define and review a resource, then "Download a template for automation"
* Existing: Resource > Export template

CLI

Create `deployment.json`

````json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": []
}
````

Create a resource group

`az group create --name learn-az204-arm --location canadacentral`

Deploy template

`az deployment group create --name emptytemplate --resource-group learn-az204-arm  --template-file "deployment.json"`

Verify deployment

Portal > Resource groups > learn-az204-arm > Overview > Deployments

Select "emptytemplate"

### Create Container Images for Solutions by using Docker

Azure Container Instances: target isolated containers, simple applications, task automation, and build jobs.

Azure Kubernetes Service (AKS): need full container orchestration, including service discovery across multiple containers, automatic scaling, and coordinated application upgrades.

Azure Container Instances can schedule both Windows and Linux containers with the same API when creating a `container group`

#### Create a Group

`az group create --name learn-deploy-aci-rg --location eastus`

#### Create a Container

`az container create --resource-group learn-deploy-aci-rg --name mycontainer --image microsoft/aci-helloworld --ports 80 --dns-name-label aci-demo-9111 --location eastus`

#### Show Container

`az container show --resource-group learn-deploy-aci-rg --name mycontainer --query "{FQDN:ipAddress.fqdn,ProvisioningState:provisioningState}" --out table`

````
FQDN                                    ProvisioningState
--------------------------------------  -------------------
aci-demo-9111.eastus.azurecontainer.io  Succeeded
````

Navigate to the container's FQDN (Fully Qualified Domain Name) to see it running.

#### Configurable Restart Policies

Containers are billed by the second, so you can specify that your containers are stopped when their processes have completed.

| Restart policy | Description |
|:---|:---|
| Always | Containers in the container group are always restarted. This policy makes sense for long-running tasks such as a web server. This is the default setting applied when no restart policy is specified at container creation. |
| Never | Containers in the container group are never restarted. The containers run one time only. |
| OnFailure | Containers in the container group are restarted only when the process executed in the container fails (when it terminates with a nonzero exit code). The containers are run at least once. This policy works well for containers that run short-lived tasks. |

Run a container to completion

`az container create --resource-group learn-deploy-aci-rg --name mycontainer-restart-demo --image microsoft/aci-wordcount:latest --restart-policy OnFailure --location eastus`

See status

`az container show --resource-group learn-deploy-aci-rg --name mycontainer-restart-demo --query containers[0].instanceView.currentState.state`

````
Result
----------
Terminated
````

See logs

`az container logs --resource-group learn-deploy-aci-rg --name mycontainer-restart-demo`

````
2020-10-23T21:05:17.6376587Z stdout F [('the', 990),
2020-10-23T21:05:17.6376587Z stdout F  ('and', 702),
2020-10-23T21:05:17.6376587Z stdout F  ('of', 628),
2020-10-23T21:05:17.6376587Z stdout F  ('to', 610),
2020-10-23T21:05:17.6376587Z stdout F  ('I', 544),
2020-10-23T21:05:17.6376587Z stdout F  ('you', 495),
2020-10-23T21:05:17.6376587Z stdout F  ('a', 453),
2020-10-23T21:05:17.6376587Z stdout F  ('my', 441),
2020-10-23T21:05:17.6376587Z stdout F  ('in', 399),
2020-10-23T21:05:17.6376587Z stdout F  ('HAMLET', 386)]
````

#### Set Environment Variables

````ps
> $env:COSMOS_DB_NAME='aci-cosmos-db-handsomeb42'

> $env:`COSMOS_DB_NAME`

aci-cosmos-db-handsomeb42

> $env:COSMOS_DB_ENDPOINT=$(az cosmosdb create --resource-group learn-deploy-aci-rg --name $env:COSMOS_DB_NAME --query documentEndpoint --output tsv)

````

`$COSMOS_DB_NAME` specifies the unique database name

`COSMOS_DB_ENDPOINT` is the endpoint address for the database

Get keys to Cosmos Database

`$env:COSMOS_DB_MASTERKEY=$(az cosmosdb keys list --resource-group learn-deploy-aci-rg --name $env:COSMOS_DB_NAME --query primaryMasterKey --output tsv)`

#### Deploy a container that works with your database

`az container create --resource-group learn-deploy-aci-rg --name aci-demo --image microsoft/azure-vote-front:cosmosdb --ip-address Public --location eastus --environment-variables COSMOS_DB_ENDPOINT=$env:COSMOS_DB_ENDPOINT COSMOS_DB_MASTERKEY=$env:COSMOS_DB_MASTERKEY`

The `--environment-variables` argument passes values to the container as environment variables when it starts.

Get the container's public IP address

`az container show --resource-group learn-deploy-aci-rg --name aci-demo --query ipAddress.ip --output tsv`

Browse to the container's IP address

#### Secured Environment Variables

By default environment variables are accessible through the Azure portal and command-line tools in plain text.

Current Behaviour:

`az container show --resource-group learn-deploy-aci-rg --name aci-demo --query containers[0].environmentVariables`

````
Name                 Value
-------------------  ----------------------------------------------------------------------------------------
COSMOS_DB_ENDPOINT   https://aci-cosmos-db-handsomeb42.documents.azure.com:443/
COSMOS_DB_MASTERKEY  DGJZClvFEZxaFv...erQhrgCPWDaaAYjYO7jWaZqxAQ==
````

Use the `--secure-environment-variables` argument instead of the `--environment-variables` argument.

`az container create --resource-group learn-deploy-aci-rg --name aci-demo-secure --image microsoft/azure-vote-front:cosmosdb --ip-address Public --location eastus --secure-environment-variables COSMOS_DB_ENDPOINT=$env:COSMOS_DB_ENDPOINT COSMOS_DB_MASTERKEY=$env:COSMOS_DB_MASTERKEY`

View environment variables

`az container show --resource-group learn-deploy-aci-rg --name aci-demo-secure --query containers[0].environmentVariables`

````
Name
-------------------
COSMOS_DB_ENDPOINT
COSMOS_DB_MASTERKEY
````

#### Use Data Volumes

By default, Azure Container Instances are stateless. If the container crashes or stops, all of its state is lost.

Persist state beyond the lifetime of the container by mounting a volume from an external store.

Create an Azure file share

````
> $env:STORAGE_ACCOUNT_NAME='mystorageaccount736475'

> az storage account create --resource-group learn-deploy-aci-rg --name $env:STORAGE_ACCOUNT_NAME --sku Standard_LRS --location eastus

> $env:AZURE_STORAGE_CONNECTION_STRING=$(az storage account show-connection-string --resource-group learn-deploy-aci-rg --name $env:STORAGE_ACCOUNT_NAME --output tsv)

> $env:AZURE_STORAGE_CONNECTION_STRING

DefaultEndpointsProtocol=https;EndpointSuffix=core.windows.net;AccountName=mystorageaccount736475;AccountKey=hmk7OMPLsNR6nJpDdXxq7O5pklqMg8Krc0Zj2kJEvKcu9/0vyOB83d88WS1RJEADwV2JwtKoNOaFh+BO+wnz7A==
````

`AZURE_STORAGE_CONNECTION_STRING` is a special environment variable that's understood by the Azure CLI.

`az storage share create --name aci-share-demo`

Get Storage Account credentials

`$env:STORAGE_KEY=$(az storage account keys list --resource-group learn-deploy-aci-rg --account-name $env:STORAGE_ACCOUNT_NAME --query "[0].value" --output tsv)`

Deploy a container and mount the file share

`az container create --resource-group learn-deploy-aci-rg --name aci-demo-files --image microsoft/aci-hellofiles --location eastus --ports 80 --ip-address Public --azure-file-volume-account-name $env:STORAGE_ACCOUNT_NAME --azure-file-volume-account-key $env:STORAGE_KEY --azure-file-volume-share-name aci-share-demo --azure-file-volume-mount-path /aci/logs/`

Get container's public IP

`az container show --resource-group learn-deploy-aci-rg --name aci-demo-files --query ipAddress.ip --output tsv`

Browse to the container's public IP and enter "Hello, World!". Submit.

````
> az storage file list -s aci-share-demo -o table

Name               Content Length    Type    Last Modified
-----------------  ----------------  ------  ---------------
1603490402419.txt  13                file

> az storage file download -s aci-share-demo -p 1603490402419.txt

> cat 1603490402419.txt

Hello, World!

````

The volume persists after this specific container is terminated and can be mounted to other containers.

#### Troubleshoot Azure Container Instances

Create a container

`az container create --resource-group learn-deploy-aci-rg --name mycontainer --image microsoft/sample-aks-helloworld --ports 80 --ip-address Public --location eastus`

Get logs from your container instance

`az container logs --resource-group learn-deploy-aci-rg --name mycontainer`

````
2020-10-23T22:05:51.2530324Z stdout F Checking for script in /app/prestart.sh
2020-10-23T22:05:51.2540316Z stdout F Running script /app/prestart.sh
2020-10-23T22:05:51.254966Z stdout F Running inside /app/prestart.sh, you could add migrations to this file, e.g.:
2020-10-23T22:05:51.254966Z stdout F
2020-10-23T22:05:51.254966Z stdout F #! /usr/bin/env bash
...

````

Get container events

The `az container attach` command provides diagnostic information during container startup. Once the container has started, it also writes standard output and standard error streams to your local terminal.

`az container attach --resource-group learn-deploy-aci-rg --name mycontainer`

````
Container 'mycontainer' is in state 'Running'...

Start streaming logs:
2020-10-23T22:05:51.2530324Z stdout F Checking for script in /app/prestart.sh
2020-10-23T22:05:51.2540316Z stdout F Running script /app/prestart.sh
2020-10-23T22:05:51.254966Z stdout F Running inside /app/prestart.sh, you could add migrations to this file, e.g.:
2020-10-23T22:05:51.254966Z stdout F
2020-10-23T22:05:51.254966Z stdout F #! /usr/bin/env bash
...
````

Execute a command in your container

`az container exec --resource-group learn-deploy-aci-rg --name mycontainer --exec-command /bin/sh`

````
# ls
__pycache__  main.py  prestart.sh  static  templates  uwsgi.ini
# exit
````

Monitor CPU and memory usage on your container

````
> $env:CONTAINER_ID=$(az container show --resource-group learn-deploy-aci-rg --name mycontainer --query id --output tsv)

> az monitor metrics list --resource $env:CONTAINER_ID --metric CPUUsage --output table

Timestamp            Name       Average
-------------------  ---------  ---------
2020-10-23 21:10:00  CPU Usage  0.0
2020-10-23 21:11:00  CPU Usage  0.0
2020-10-23 21:12:00  CPU Usage
2020-10-23 21:13:00  CPU Usage
2020-10-23 21:14:00  CPU Usage  0.0
...

> az monitor metrics list --resource $env:CONTAINER_ID --metric MemoryUsage --output table

Timestamp            Name          Average
-------------------  ------------  ----------
2020-10-23 21:10:00  Memory Usage  18759680.0
2020-10-23 21:11:00  Memory Usage  18804736.0
2020-10-23 21:12:00  Memory Usage
2020-10-23 21:13:00  Memory Usage
2020-10-23 21:14:00  Memory Usage  18923520.0
...
````

#### Create a container image for deployment to Azure Container Instances

Get the code

`git clone https://github.com/Azure-Samples/aci-helloworld.git`

Build the container image

Docker file

````dockerfile
FROM node:8.9.3-alpine
RUN mkdir -p /usr/src/app
COPY ./app/ /usr/src/app/
WORKDIR /usr/src/app
RUN npm install
CMD node /usr/src/app/index.js
````

`docker build ./aci-helloworld -t aci-tutorial-app`

````
D:\Development\00_working [master +22 ~0 -0 !]> docker build ./aci-helloworld -t aci-tutorial-app
Sending build context to Docker daemon    130kB
Step 1/6 : FROM node:8.9.3-alpine
8.9.3-alpine: Pulling from library/node
1160f4abea84: Pull complete
66ff3f133e43: Pull complete
4c8ff6f0a4db: Pull complete
Digest: sha256:40201c973cf40708f06205b22067f952dd46a29cecb7a74b873ce303ad0d11a5
Status: Downloaded newer image for node:8.9.3-alpine
 ---> 144aaf4b1367
Step 2/6 : RUN mkdir -p /usr/src/app
 ---> Running in e787f9fa5645
...
````

`docker images`

````
REPOSITORY                                 TAG                      IMAGE ID            CREATED             SIZE
aci-tutorial-app                           latest                   642c13fa3115        43 seconds ago      71.1MB
...
````

Run the container locally

````
> docker run -d -p 8080:80 aci-tutorial-app
a2e3e4435db58ab0c664ce521854c2e1a1bda88c9cf2fcff46aedf48df86cccf
````

Output from the `docker run` command displays the running container's ID if the command was succesful.

Navigate to `http://localhost:8080` in the browser to confirm the container is running.

### Publish an Image to the Azure Container Registry

#### Create an Azure container registry

Create a resource group

`az group create --name az204-learn-aci-registry --location eastus`

Create a container registry

`az acr create --resource-group az204-learn-aci-registry --name learnaz204registry --sku Basic`

Log in to container registry

`az acr login --name learnaz204registry`

Push an image to a private registry like Azure Container Registry

Tag container image

````
> az acr show --name learnaz204registry --query loginServer --output table

Result
-----------------------------
learnaz204registry.azurecr.io

> docker tag aci-tutorial-app learnaz204registry.azurecr.io/aci-tutorial-app:v1

> docker images

REPOSITORY                                       TAG                      IMAGE ID            CREATED             SIZE
aci-tutorial-app                                 latest                   642c13fa3115        41 minutes ago      71.1MB
learnaz204registry.azurecr.io/aci-tutorial-app   v1                       642c13fa3115        41 minutes ago      71.1MB
...
````

Push image to Azure Container Registry

````
> docker push learnaz204registry.azurecr.io/aci-tutorial-app:v1

The push refers to a repository [learnaz204registry.azurecr.io/aci-tutorial-app]
3db9cac20d49: Pushed
13f653351004: Pushed
4cd158165f4d: Pushed
d8fbd47558a8: Pushed
44ab46125c35: Pushed
5bef08742407: Pushed
v1: digest: sha256:ed67fff971da47175856505585dcd92d1270c3b37543e8afd46014d328f05715 size: 1576
````

List images in Azure Container Registry

````
> az acr repository list --name learnaz204registry --output table
Result
----------------
aci-tutorial-app
````

View the `tags` for a specific image

````
> az acr repository show-tags --name learnaz204registry --repository aci-tutorial-app --output table
Result
--------
v1
````

### Run Containers by using Azure Container Instance

Get registry credentials

Best practice for many scenarios is to create and configure an Azure Active Directory service principal with pull permissions to your registry.

https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-aci

````
> az acr show --name learnaz204registry --query loginServer
Result
-----------------------------
learnaz204registry.azurecr.io
````

Deploy container

`az container create --resource-group az204-learn-aci-registry --name aci-tutorial-app --image learnaz204registry.azurecr.io/aci-tutorial-app:v1 --cpu 1 --memory 1 --registry-login-server learnaz204registry.azurecr.io --registry-username <service-principal-ID> --registry-password <service-principal-password> --dns-name-label <aciDnsLabel> --ports 80`

Verify deployment progress

`az container show --resource-group az204-learn-aci-registry --name aci-tutorial-app --query instanceView.state`

View the application and container logs

`az container show --resource-group az204-learn-aci-registry --name aci-tutorial-app --query ipAddress.fqdn`

Navigate to the displayed DNS name to see the running application.

View logs

`az container logs --resource-group az204-learn-aci-registry --name aci-tutorial-app`

````
listening on port 80
::ffff:10.240.0.4 - - [21/Jul/2017:06:00:02 +0000] "GET / HTTP/1.1" 200 1663 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 Safari/537.36"
::ffff:10.240.0.4 - - [21/Jul/2017:06:00:02 +0000] "GET /favicon.ico HTTP/1.1" 404 150 "http://aci-demo.eastus.azurecontainer.io/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 Safari/537.36"
````

#### Container Groups

A container group is a collection of containers that get scheduled on the same host machine. The containers in a container group share a lifecycle, resources, local network, and storage volumes. It's similar in concept to a pod in Kubernetes.

Deployment

* Resource Manager template: when you need to deploy additional Azure service resources (ex:, an Azure Files share) when you deploy the container instances.
* YAML file: recommended when your deployment includes only container instances.

Resource allocation

Azure Container Instances allocates resources such as CPUs, memory, and optionally GPUs (preview) to a multi-container group by adding the resource requests of the instances in the group. Taking CPU resources as an example, if you create a container group with two container instances, each requesting 1 CPU, then the container group is allocated 2 CPUs.

Networking

Container groups can share an external-facing IP address, one or more ports on that IP address, and a DNS label with a fully qualified domain name (FQDN).

Within a container group, container instances can reach each other via localhost on any port, even if those ports aren't exposed externally on the group's IP address or from the container.

Storage

* Azure file share
* Secret
* Empty directory
* Cloned git repo

Common Scenarios

Multi-container groups are useful in cases where you want to divide a single functional task into a small number of container images.

Examples:

* A front-end container and a back-end container. The front end might serve a web application, with the back end running a service to retrieve data.
* An application container and a logging container. The logging container collects the logs and metrics output by the main application and writes them to long-term storage.

## Create Azure App Service Web Apps

* Azure App Service for hosting web apps, mobile app back ends, RESTful APIs, or automated business processes
* No need to manage VMs or networking resources
* Multiple languages and frameworks
* Containerization and Docker
* DevOps optimization (CI/CD)

App Service can host web apps natively on Linux for supported application stacks.

* Windows web apps
* Linux web apps
* Docker containers
* Mobile apps
* Functions

### Create an Azure App Service Web App

````ps
mkdir hellodotnetcore
cd hellodotnetcore

dotnet new web

dotnet run
````

Initialize a Git repository for the .NET Core project

````ps
git init
git add .
git commit -m "init commit"
````

Configure a deployment user

`az webapp deployment user set --user-name <username> --password <password>`

Create a resource group

`az group create --name az204learnappservice --location canadacentral`

Create an Azure App Service plan

`az appservice plan create --name azure204learnappserviceplan --resource-group az204learnappservice --sku F1 --is-linux`

Create a web app

`az --% webapp create --resource-group az204learnappservice --plan azure204learnappserviceplan --name webapp-learn-azure204 --runtime "DOTNETCORE|3.1" --deployment-local-git`

Creates an empty web app in a Linux container, with git deployment enabled.

`https://webapp-learn-azure204.azurewebsites.net`

Push to Azure from Git

````ps
git remote add azure https://handsomeb@webapp-learn-azure204.scm.azurewebsites.net/webapp-learn-azure204.git`
git push azure master
````

Browse to the app

`https://webapp-learn-azure204.azurewebsites.net`

Update and redeploy the code

Edit `Startup.cs` to change the response message.

`await context.Response.WriteAsync("Hello Azure!");`

Commit changes in Git then push code changes to Azure

````ps
git commit -am "update to app"
git push azure master
````

Browse to the app and see changes

### Enable Diagnostics Logging

Azure provides built-in diagnostics to assist with debugging an App Service app.

* enable diagnostic logging
* add instrumentation to an application
* access the information logged by Azure

You can also send logs to Azure Monitor

* Application logging (Windows, Linux): log messages generated by application code
* Web server logging (Windows): raw HTTP request data.
* Detailed Error Messages (Windows): copies of `.htm` error pages as would be sent to client browser (should not be delivered in production).
* Failed request tracing (Windows): detailed tracing information on failed requests, including IIS pipeline.
* Deployment logging (Windows, Linux): logs from published content.

#### Enable application logging (Windows)

Windows App > App Service logs

Select On for either Application Logging (Filesystem - short term < 12 hours) or Application Logging (Blob - longer term to blob container), or both.

Save

#### Enable application logging (Linux/Container)

Linux App > App Service logs

Application logging > File System

* Quota (MB)
* Retention Period (Days)

Save

#### Enable web server logging (Windows)


#### Log detailed errors (Windows)

App ServiceLogs > Detailed Error Logging > On

App ServiceLogs > Failed Request Tracing > On

Save

#### Add log messages in code

`System.Diagnostics.Trace.TraceError("If you're seeing this, something bad happened");`

#### Stream logs

Enable the log type that you want

`az webapp log tail --name appname --resource-group myResourceGroup`

Filter specific events, like __Error__

`az webapp log tail --name appname --resource-group myResourceGroup --filter Error`

Filter specific log types, like __HTTP__

`az webapp log tail --name appname --resource-group myResourceGroup --path http`

#### Access log files

* Linux/container apps: https://<app-name>.scm.azurewebsites.net/api/logs/docker/zip
* Windows apps: https://<app-name>.scm.azurewebsites.net/api/dump

#### Send logs to Azure Monitor (preview)

Diagnostic settings > Add diagnostic setting

Query logs with Azure Monitor

### Deploy Code to a Web App

#### Deploy via git commits to Azure (see above)

#### Deploy with a ZIP or WAR file

Create a project ZIP file of the `dotnet publish` output folder

`Compress-Archive -Path * -DestinationPath <file-name>.zip`

Deploy ZIP file

navigate to `https://<app_name>.scm.azurewebsites.net/ZipDeployUI`

CLI

`az webapp deployment source config-zip --resource-group <group-name> --name <app-name> --src clouddrive/<filename>.zip`

#### Deploy ZIP file with REST APIs

cURL

`curl -X POST -u <deployment_user> --data-binary @"<zip_file_path>" https://<app_name>.scm.azurewebsites.net/api/zipdeploy`

PowerShell

`Publish-AzWebapp -ResourceGroupName <group-name> -Name <app-name> -ArchivePath <zip-file-path>`

#### Avoid locking failures during deployment

* Run your app from the ZIP package directly without unpacking it.
* Stop your app or enable offline mode for your app during deployment. For more information, see Deal with locked files during deployment.
* Deploy to a staging slot with auto swap enabled.

### Configure Web App Settings including SSL, API and Connection Strings

Replace the need for web.config transforms

#### Configure app settings

App Services > App > Configuration >

Encrypted at rest; transmitted over an encrypted channel

* Application settings
    * non-.Net get access via environment variables (KVP)
* Connection strings
    * connection string in azure replaces local development value of same name

At runtime, connection strings are available as environment variables, prefixed with the following connection types:

* SQLServer: SQLCONNSTR_
* MySQL: MYSQLCONNSTR_
* SQLAzure: SQLAZURECONNSTR_
* Custom: CUSTOMCONNSTR_
* PostgreSQL: POSTGRESQLCONNSTR_

__CLI__

list

`az webapp config appsettings list --name webapp-learn-azure204 --resource-group az204learnappservice`

`az webapp config connection-string list --name webapp-learn-azure204 --resource-group az204learnappservice`

assign

`az webapp config appsettings set --name <app-name> --resource-group <resource-group-name> --settings <setting-name>="<value>"`

remove

`az webapp config appsettings delete --name <app-name> --resource-group <resource-group-name> --setting-names {<names>}`

Edit in bulk

Advanced edit > Update

#### Configure general settings

* Stack settings: The software stack to run the app, including the language and SDK versions.
* Platform settings: Lets you configure settings for the hosting platform like bitness (32/64) websocket protocol, Always on, managed pipeline version, http version, debugging

#### Configure default documents

Windows only  

#### Configure path mappings

* Windows apps (uncontainerized): IIS handlers
* Containerized apps: add custom storage for a containerized app

#### Custom Domain Names

* Purchase a domain
* Assign hostnames to app
* Renew the domain
* Manage custom DNS records
* Direct default URL to a custom directory (other than root of app code)

#### Add a TLS/SSL certificate in Azure App Service

* Create a free App Service Managed Certificate (Preview): secure www of custom domain or any non-naked domain in App Service
    * no wildcards
    * no naked domains
    * not exportable
    * no supported in App Service Environment
    * does not support A records (ex: automatic renewal doesn't work with A Records)
* Purchase an App Service certificate: a private certification managed by Azure
    * handles purchase, domain verifications, Key Vault storage, renewal
* Import a certificate from Key Vault: Useful if you use Azure Key Vault to manage your PKCS12 certificates
* Upload a private certificate: If you already have a private certificate from a third-party provider
    * must be password-protected PFX file
    * private key at least 2048 bite long
    * contains all intermediate certificates in the certificate chain (Merge intermediate certificates in order before exporting)
    * `openssl pkcs12 -export -out myserver.pfx -inkey <private-key-file> -in <merged-certificate-file>`
* Upload a public certificate (.cer format): If you need a certificate to access remote resources.

#### Secure a custom DNS name with a TLS/SSL binding in Azure App Service

* Add a private certificate to App Service that satisfies all the private certificate requirements.
* Create a TLS binding to the corresponding custom domain.

App Services > app-name > TLS/SSL Binding

Custom domains > Add binding

TLS/SSL settings > Add TLS/SSL binding

Upload PFX Certificate or Import App Service Certificate

Create binding

* Custom domain: The domain name to add the TLS/SSL binding for.
* Private Certificate Thumbprint: The certificate to bind.
* TLS/SSL Type:
    * SNI SSL - Multiple SNI SSL bindings may be added
    * IP SSL - Only one IP SSL binding may be added.

Test HTTPS in browsers

Prevent IP changes

1. Upload the new certificate.
2. Bind the new certificate to the custom domain you want without deleting the old one. This action replaces the binding instead of removing the old one.
3. Delete the old certificate.

Enforce HTTPS

App Services > app-name > TLS/SSL settings > Bindings > HTTPS Only > On

Enforce TLS versions

TLS 1.2 is the default. Set the minimum TLS version:

App Services > app-name > TLS/SSL settings > Bindings > Minimum TLS Version > 1.0 or 1.1 or 1.2

Handle TLS termination

In App Service, TLS termination happens at the network load balancers, so all HTTPS requests reach your app as unencrypted HTTP requests. If your app logic needs to check if the user requests are encrypted or not, inspect the `X-Forwarded-Proto` header.


#### Add CORS functionality

App Service has built-in support for Cross-Origin Resource Sharing (CORS) for RESTful APIs.

````
No 'Access-Control-Allow-Origin' header is present on the requested resource
````

A domain mismatch between the browser app and a remote resource, and the fact that the API in App Service is not sending the Access-Control-Allow-Origin header, makes the browser prevent cross-domain content from loading.

`az webapp cors add --resource-group myResourceGroup --name <app-name> --allowed-origins 'http://localhost:5000'`

You can use your own CORS utilities instead of App Service CORS for more flexibility. For example, you may want to specify different allowed origins for different routes or methods. Since App Service CORS lets you specify one set of accepted origins for all API routes and methods, you would want to use your own CORS code.

__Don't try to use App Service CORS and your own CORS code together. When used together, App Service CORS takes precedence and your own CORS code has no effect.__

### Implement autoscaling rules, including scheduled autoscaling, and scaling by operational or system metrics

* Scale Up: add more CPU, memory, disk space or extra features. Change the pricing tier of the App Service plan that the app belongs to

* Scale out: increase the number of VM instances running the app. Scale out to as many as 30 instances, depending on pricing tier. __Isolated__ tier increase scale-out count to 100.

#### Common Autoscale Patterns

* Scale based on CPU:
    * You want to scale out/scale in based on CPU
    * You want to ensure there is a minimum number of instances.
    * You want to ensure that you set a maximum limit to the number of instances you can scale to.

* Scale differently on weekdays vs weekends
    * You want 3 instances by default (on weekdays)
    * You don't expect traffic on weekends and hence you want to scale down to 1 instance on weekends.

* Scale differently during holidays
    * You want to scale up/down based on CPU usage by default
    * However, during holiday season (or specific days that are important for your business) you want to override the defaults and have more capacity at your disposal.

* Scale based on custom metric
    * You want to scale the API tier based on custom events in the front end (example: You want to scale your checkout process based on the number of items in the shopping cart)

#### Create an Autoscale Setting 

Add/remove instances of service based on preset conditions. Create rules via portal.

Monitor > Autoscale > Enable Autoscale

Configure default profile

1. Provide a Name for the autoscale setting.
2. In the default profile, ensure the Scale mode is set to 'Scale to a specific instance count'.
3. Set the instance count to 1. This setting ensures that when no other profile is active, or in effect, the default profile returns the instance count to 1.

#### Create recurrence profile

Add a scale condition > Name 

* Scale mode = 'Scale based on a metric'
* Instance limits:
    * *Minimum = '1'
    * Maximum = '2'
    * Default = '1'

Create a scale-out rule

Add a rule: "When the total requests exceeds 10 in 5 minutes or less, add one additional instance of the Web App to the App Service Plan"

* Metric source = 'other resource'
* Resource type = 'App Services'
* Resource = the Web App
* Time aggregation = 'Total'
* Metric = 'Requests'
* Time grain statistic = 'Sum'
* Operator = 'Greater than'
* Threshold = '10'
* Duration = '5' minutes
* Operation = 'Increase count by'
* Instance count = '1'
* Cool down = '5' minutes

Cool Down prevents too many scaling requests while waiting for any previous operations to be completed. A 5 minute cool down means this rule cannot fire more than once every 5 minutes.

#### Create a scale-in rule

Always have a scale-in rule to accompany a scale-out rule to prevent over-provisioning of resources.

Add a rule: "When there is less than an average of 5 requests in 5 minutes decrease the instance count by 1.

* Metric source = 'other resource'
* Resource type = 'App Services'
* Resource = the Web App
* Time aggregation = 'Total'
* Metric = 'Requests'
* Time grain statistic = 'Average'
* Operator = 'Less than'
* Threshold = '5'
* Duration = '5' minutes
* Operation = 'Decrease count by'
* Instance count = '1'
* Cool down = '10' minutes

Scale-in cool down should be more conservative than scale out, so longer (better to have excess capacity than be under-provisioned).

#### Scale based on a schedule

Add a scale condition > Repeat specific days

Select the days and the start/end time for when the scale condition should be applied.

#### Disable Autoscale and manually scale your instances

Disable autoscale

You can now set the number of instances that you want to scale to manually.

## Implement Azure Functions

A managed Functions-as-a-Service (FaaS) environment. Run small pieces of code without worrying about application infrastructure.

### Implement Input and Output Bindings for a Function

* Triggers cause a function to run
* A function must have exactly one trigger
* Triggers have associated data provided as the payload to the function
* Binding declaratively connects another resource to the function
    * input binding
    * output binding
    * both
* Binding data is provided to the function as parameters

#### Examples:

1. A new queue message arrives which runs a function to write to another queue.
2. A scheduled job reads Blob Storage contents and creates a new Cosmos DB document.
3. The Event Grid is used to read an image from Blob Storage and a document from Cosmos DB to send an email.
4. A webhook that uses Microsoft Graph to update an Excel sheet.


#### Answers (Trigger, Input binding, Output binding):

1. Queue, none, Queue
2. I: Timer, Blob Storage, Cosmos DB
3. Event Grid, Blob storage and Cosmos DB, SendGrid
4. HTTP, none, MS Graph

* In a C# class library function triggers and bindings are defined by decorating methods and parameters with C# attributes
* In all other implementations the `function.json` file is updated
* Triggers and binding have a `direction` property
    * Triggers: always `in`
    * Input and Output bindings: `in` and `out`
    * Some bindings support `inout` (some Blob types)
    * When you use attributes in a class libary the direction is provided in an attribute constructor or inferred from the parameter type

Example:  Write a new row to Azure Table storage whenever a new message appears in Azure Queue storage.

```json
{
  "bindings": [
    {
      "type": "queueTrigger",
      "direction": "in",
      "name": "order",
      "queueName": "myqueue-items",
      "connection": "MY_STORAGE_ACCT_APP_SETTING"
    },
    {
      "type": "table",
      "direction": "out",
      "name": "$return",
      "tableName": "outTable",
      "connection": "MY_TABLE_STORAGE_ACCT_APP_SETTING"
    }
  ]
}
```

### Implement Function Triggers by Using Data Operations, Timers, and Webhooks

https://docs.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings?tabs=csharp

#### Data Operations

* Queue
* Blob Storage
* Azure Cosmos DB
* Event Hubs
* ...

Blob storage

| Action | Type |
|---|---|
| Run a function as blob storage data changes | Trigger |
| Read blob storage data in a function | Input binding |
| Allow a function to write blob storage data | Output binding |

Trigger

````c#
[FunctionName("BlobTriggerCSharp")]        
public static void Run([BlobTrigger("samples-workitems/{name}")] Stream myBlob, string name, ILogger log)
{
    log.LogInformation($"C# Blob trigger function Processed blob\n Name:{name} \n Size: {myBlob.Length} Bytes");
}
````

Input

````c#
[FunctionName("BlobInput")]
public static void Run(
    [QueueTrigger("myqueue-items")] string myQueueItem,
    [Blob("samples-workitems/{queueTrigger}", FileAccess.Read)] Stream myBlob,
    ILogger log)
{
    log.LogInformation($"BlobInput processed blob\n Name:{myQueueItem} \n Size: {myBlob.Length} bytes");
}
````

Output

Function triggered by the creation of an image blob in the `sample-images container`. It creates small and medium size copies of the image blob.

````c#
 [FunctionName("ResizeImage")]
public static void Run([BlobTrigger("sample-images/{name}")] Stream image,
    [Blob("sample-images-sm/{name}", FileAccess.Write)] Stream imageSmall,
    [Blob("sample-images-md/{name}", FileAccess.Write)] Stream imageMedium)
{
  ...
````

#### Timer

NCRONTAB schedule

`{second} {minute} {hour} {day} {month} {day-of-week}`

| Example | When triggered |
|---|---|
| "0 */5 * * * *" | once every five minutes |
| "0 0 * * * *" | once at the top of every hour |
|"0 0 9-17 * * *" | once every hour from 9 AM to 5 PM|
|"0 30 9 * * *" | at 9:30 AM every day|
|"0 30 9 * * 1-5" | at 9:30 AM every weekday|
|"0 30 9 * Jan Mon" | at 9:30 AM every Monday in January|

````c#
[FunctionName("TimerTriggerCSharp")]
public static void Run([TimerTrigger("0 */5 * * * *")]TimerInfo myTimer, ILogger log)
{
    if (myTimer.IsPastDue)
    {
        log.LogInformation("Timer is running late!");
    }
    log.LogInformation($"C# Timer trigger function executed at: {DateTime.Now}");
}
````

#### Webhook

* Http (trigger)
* "user-defined HTTP callbacks" (output binding)

````c#
[FunctionName("HttpTriggerCSharp")]
public static async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)]
    HttpRequest req, ILogger log)
{
    log.LogInformation("C# HTTP trigger function processed a request.");

    string name = req.Query["name"];

    string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
    dynamic data = JsonConvert.DeserializeObject(requestBody);
    name = name ?? data?.name;

    return name != null
        ? (ActionResult)new OkObjectResult($"Hello, {name}")
        : new BadRequestObjectResult("Please pass a name on the query string or in the request body");
}
````

````json
{
    "extensions": {
        "http": {
            "routePrefix": "api",
            "maxOutstandingRequests": 200,
            "maxConcurrentRequests": 100,
            "dynamicThrottlesEnabled": true,
            "hsts": {
                "isEnabled": true,
                "maxAge": "10"
            },
            "customHeaders": {
                "X-Content-Type-Options": "nosniff"
            }
        }
    }
}
````

### Implement Azure Durable Functions

Durable Functions is an extension of Azure Functions that lets you write stateful functions in a serverless compute environment

* stateful workflows: orchestrator functions (coordinate the execution of other Durable functions within a function app)
* stateful entities: entity functions (reading and updating small pieces of state)

The primary use case for Durable Functions is simplifying complex, stateful coordination requirements in serverless applications.

#### Function Chaining

Asequence of functions executes in a specific order. In this pattern, the output of one function is applied to the input of another function.

````c#
[FunctionName("Chaining")]
public static async Task<object> Run(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    try
    {
        var x = await context.CallActivityAsync<object>("F1", null);
        var y = await context.CallActivityAsync<object>("F2", x);
        var z = await context.CallActivityAsync<object>("F3", y);
        return  await context.CallActivityAsync<object>("F4", z);
    }
    catch (Exception)
    {
        // Error handling or compensation goes here.
    }
}
````

You can use the `context` parameter to invoke other functions by name, pass parameters, and return function output. Each time the code calls `await`, the Durable Functions framework checkpoints the progress of the current function instance. If the process or virtual machine recycles midway through the execution, the function instance resumes from the preceding `await` call.

#### Fan-out/Fan-in

Execute multiple functions in parallel and then wait for all functions to finish.

````c#
[FunctionName("FanOutFanIn")]
public static async Task Run(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var parallelTasks = new List<Task<int>>();

    // Get a list of N work items to process in parallel.
    object[] workBatch = await context.CallActivityAsync<object[]>("F1", null);
    for (int i = 0; i < workBatch.Length; i++)
    {
        Task<int> task = context.CallActivityAsync<int>("F2", workBatch[i]);
        parallelTasks.Add(task);
    }

    await Task.WhenAll(parallelTasks);

    // Aggregate all N outputs and send the result to F3.
    int sum = parallelTasks.Sum(t => t.Result);
    await context.CallActivityAsync("F3", sum);
}
````

The Fan-out can be accomplished with a queue and normal functions, but the fan-in can be challenging (track when the queue-triggered functions end, and then store function outputs.)

The fan-out work is distributed to multiple instances of the `F1` function. The work is tracked by using a dynamic list of tasks. `Task.WhenAll` is called to wait for all the called functions to finish. Then, the `F2` function outputs are aggregated from the dynamic task list and passed to the `F3` function.

The automatic checkpointing that happens at the await call on `Task.WhenAll` ensures that a potential midway crash or reboot doesn't require restarting an already completed task.

#### Async HTTP APIs

Addresses the problem of coordinating the state of long-running operations with external clients. A common way to implement this pattern is by having an HTTP endpoint trigger the long-running action. Then, redirect the client to a status endpoint that the client polls to learn when the operation is finished.

Durable Functions provides built-in support for this pattern, exposing built-in HTTP APIs that manage long-running orchestrations.

#### Monitoring

The monitor pattern refers to a flexible, recurring process in a workflow. An example is polling until specific conditions are met. You can use a regular timer trigger to address a basic scenario, such as a periodic cleanup job, but its interval is static and managing instance lifetimes becomes complex.

Durable Functions can implement multiple monitors that observe arbitrary endpoints. The monitors can end execution when a condition is met, or another function can use the durable orchestration client to terminate the monitors. You can change a monitor's wait interval based on a specific condition (for example, exponential backoff.)


#### Human interaction

Automated processes involving human interaction are not highly available and as responsive as cloud services. An automated process might allow for this interaction by using timeouts and compensation logic.

You can use a durable timer and a branching process that waits for an external event

External events are one-way asynchronous operations. They are not suitable for situations where the client sending the event needs a synchronous response from the orchestrator function.

#### Aggregator (stateful entities)

Aggregating event data over a period of time into a single, addressable entity.

The aggregator might need to take action on event data as it arrives, and external clients may need to query the aggregated data.

 Implement this pattern with normal, stateless functions makes concurrency control a huge challenge.
 
 * multiple threads modifying the same data at the same time
 * the aggregator only runs on a single VM at a time.

 Durable Entities:

 ````c#
 [FunctionName("Counter")]
public static void Counter([EntityTrigger] IDurableEntityContext ctx)
{
    int currentValue = ctx.GetState<int>();
    switch (ctx.OperationName.ToLowerInvariant())
    {
        case "add":
            int amount = ctx.GetInput<int>();
            ctx.SetState(currentValue + amount);
            break;
        case "reset":
            ctx.SetState(0);
            break;
        case "get":
            ctx.Return(currentValue);
            break;
    }
}
 ````

#### Create a Durable Function

VS 2019 > New > Project > Azure Functions > Empty

Add > New Azure Function > Azure Function > Add

Durable Functions Orchestration > OK


````c#
public static class Function1
{
    [FunctionName("Function1")]
    public static async Task<List<string>> RunOrchestrator(
        [OrchestrationTrigger] IDurableOrchestrationContext context)
    {
        var outputs = new List<string>();

        // Replace "hello" with the name of your Durable Activity Function.
        outputs.Add(await context.CallActivityAsync<string>("Function1_Hello", "Tokyo"));
        outputs.Add(await context.CallActivityAsync<string>("Function1_Hello", "Seattle"));
        outputs.Add(await context.CallActivityAsync<string>("Function1_Hello", "London"));

        // returns ["Hello Tokyo!", "Hello Seattle!", "Hello London!"]
        return outputs;
    }

    [FunctionName("Function1_Hello")]
    public static string SayHello([ActivityTrigger] string name, ILogger log)
    {
        log.LogInformation($"Saying hello to {name}.");
        return $"Hello {name}!";
    }

    [FunctionName("Function1_HttpStart")]
    public static async Task<HttpResponseMessage> HttpStart(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post")] HttpRequestMessage req,
        [DurableClient] IDurableOrchestrationClient starter,
        ILogger log)
    {
        // Function input comes from the request content.
        string instanceId = await starter.StartNewAsync("Function1", null);
        log.LogInformation($"Started orchestration with ID = '{instanceId}'.");
        return starter.CreateCheckStatusResponse(req, instanceId);
    }
}
````
