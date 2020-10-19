# Connect to and Consume Azure services and Third-party Services (25-30%)

## Develop an App Service Logic App

Logic Apps schedule, orchestrate and automate tasks and workflows like application and data integrations. The use a trigger that fires based on a specified event (this could be a schedule). They can be built with the visual Logic Apps Designer, editing JSON Objects, or using PowerShell to tie together existing components (or writing custom code).

Combine:

* BizTalk
* Azure Functions
* Service Bus
* API Management

Uses sets of connectors for common systems and applications

Helps "glue" and connect disparate systems together

### Create a Logic App

1. Find and configure a trigger
2. Add one or more steps
3. Run the logic app

...or you can use existing templates that target common areas and workflows (example: delete old azure blobs "once a day scan a storage account and delete all blobs older than a defined value")

You can also create logic apps using Visual Studio using the Azure Logic Apps Tools. __Requires internet access using the embedded Logic App Designer for creating resources and reading properties from connectors__

You can us VS Code to manage logic app workflow definitions (i.e. edit the JSON definitions).

* download the VS Code Azure Logic Apps extension
* sign into Azure
* Access the Logic Apps area and create a new logic app
* Edit the `(.logicapp.json) file` definition file
* after editing you can review in a read-only design view with "open in designer"
* You can access the logic app via the portal, enable or disable your app and also edit the workflow with VS Code

### Create a custom connector for Logic Apps

Create a "Logic Apps Custom Connector" before using it in a Logic App

You can define the connector manual, using an OpenAPI definition or Import a Postman collection.

### Create a custom template for Logic Apps

Azure Logic Apps provides a prebuilt logic app Azure Resource Manager (ARM) template that you can reuse for creating logic apps and to define the resources and parameters to use for deployment

Can create parameterized logic app templates with:

* Visual Studio
* PowerShell using the `LogicAppTemplate` module

Defines the infrastructure, resources, parameters, and other information for provisioning and deploying your logic app.

example: template parameters can be used to define connection strings for dev, QA and prod environments.

A Resource Manager template at the highest level:

```json
{
   "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
   "contentVersion": "1.0.0.0",
   "parameters": {},
   "variables": {},
   "functions": [],
   "resources": [],
   "outputs": {}
}
```

* Logic app template file name: `<logic-app-name>.json`
* Parameters file name: `<logic-app-name>.parameters.json`

template parameters for sensitive values uses the securestring or secureobject parameter types to reference the key, with the related value then stored in Azure Key Vault.

## Implement API management

* API Management (APIM) is a way to create consistent and modern API gateways for existing back-end services.

Components

* API gateway endpoint
   * Accepts API calls and routes to backend services.
   * Verifies API keys, JWT tokens, certificates, and other credentials.
   * Enforces usage quotas and rate limits.
   * Caches backend responses
   * Logs metadata for analytics

* Azure portal is the administrative interface
   * Define or import API schema.
   * Package APIs into products.
   * Set up policies like quotas or transformations on the APIs.
   * Get insights from analytics.
   * Manage users.

* Developer portal
   * Read API documentation.
   * Try out an API via the interactive console.
   * Create an account and subscribe to get API keys.
   * Access analytics on their own usage.

APIs contain operations that map to backend services

Products surface APIs to developers (one or more APIs)

Groups are used to manage the visibility of products to developers

* Administrators
* Developers
* Guests
* use AD groups
* create custom groups

Developers represent the user accounts in an API Management service instance

Policies are a collection of statements that are executed sequentially on the request or response of an API

### create an API Management (APIM) instance

* Azure Portal
* CLI
* PowerShell
* VS Code (Azure API Management Extension - Preview)
* ARM Template (Microsoft.ApiManagement service template reference)

#### Azure Portal

New Resource > Integration > API Management

* Wait for the APIM instance to be provisioned (this can take 30-60 min)

APIs > OpenAPI > Full

* Once you import the backend API into API Management, your API Management API becomes a façade for the backend API

#### CLI

Create a resource group

`az group create --name myResourceGroup --location centralus`

Create a new service

````
az apim create --name myapim --resource-group myResourceGroup \
  --publisher-name Contoso --publisher-email admin@contoso.com \
  --no-wait
````

check status of deployment

`az apim show --name myapim --resource-group myResourceGroup --output table`

When your API Management service instance is online, you're ready to use it.

#### PowerShell

Create a resource group

`New-AzResourceGroup -Name myResourceGroup -Location WestUS`

Create a new service

````ps
New-AzApiManagement -Name "myapim" -ResourceGroupName "myResourceGroup" `
  -Location "West US" -Organization "Contoso" -AdminEmail "admin@contoso.com" 
````

check status of deployment

`Get-AzApiManagement -Name "myapim" -ResourceGroupName "myResourceGroup"`

When your API Management service instance is deployed, you're ready to use it.

#### Test the new API in the Azure portal

APIs > Demo Conference API > Test tab > GetSpeakers

Enter parameters (if any), click "Send"

#### Create and publish a product

Products > Add > Create

select product

APIs > Add

#### Mock API responses

APIs > Add Blank

Add operation > GET

Add a response (encoding, value)

Enable Mocking via inbound processing rule

#### Configure alerts based on metrics and activity logs

Based on triggers (like unauthorized request) you can:

* Send an email notification
* Call a webhook
* Invoke an Azure Logic App

### configure authentication for APIs

API Management provides the capability to secure access to APIs using client certificates.

* validate incoming certificate
* check certificate properties against desired values using policy expressions

To receive and verify client certificates over HTTP/2 in the Developer, Basic, Standard, or Premium tiers you must turn on the "Negotiate client certificate"

#### Checking the issuer and subject

````xml
<choose>
    <when condition="@(context.Request.Certificate == null || !context.Request.Certificate.Verify() || context.Request.Certificate.Issuer != "trusted-issuer" || context.Request.Certificate.SubjectName.Name != "expected-subject-name")" >
        <return-response>
            <set-status code="403" reason="Invalid client certificate" />
        </return-response>
    </when>
</choose>
````

#### Checking the thumbprint

````xml
<choose>
    <when condition="@(context.Request.Certificate == null || !context.Request.Certificate.Verify() || context.Request.Certificate.Thumbprint != "DESIRED-THUMBPRINT-IN-UPPER-CASE")" >
        <return-response>
            <set-status code="403" reason="Invalid client certificate" />
        </return-response>
    </when>
</choose>
````

#### Checking a thumbprint against certificates uploaded to API Management

````xml
<choose>
    <when condition="@(context.Request.Certificate == null || !context.Request.Certificate.Verify()  || !context.Deployment.Certificates.Any(c => c.Value.Thumbprint == context.Request.Certificate.Thumbprint))" >
        <return-response>
            <set-status code="403" reason="Invalid client certificate" />
        </return-response>
    </when>
</choose>
````

#### Upload and use a certificate

Alternative to using Azure Key Vault

APIM > Security > Certificates > Add > Create

APIs > API Management > Design > Backend > Gateway credentials

### define policies for APIs

Policies are a collection of Statements that are executed sequentially on the request or response of an API.

Policies are applied inside the gateway which sits between the API consumer and the managed API.

#### Policy organization

* inbound
* backend
* outbound
* on-error

#### Policy scopes (in order)

1. Global scope: All APIs > inbound/outbound processing
2. Product scope: Products > Policies
3. API scope: `api-name` > All operations > inbound/outbound processing
4. Operation scope: `api-name` > `oepration-name` > inbound/outbound processing

````xml
<policies>
   <inbound>
      <cross-domain />
      <base />
      <find-and-replace from="xyz" to="abc" />
      <ip-filter action="allow">
         <address>1.2.3.4</address>
      </ip-filter>
   </inbound>
   ...
</policies>
````

#### Transform an API

* Strip response headers

* Replace original URLs in the body of the API response with APIM gateway URLs

APIs > Demo Conference API > Test > GetSpeakers > Send

````json
...
request-context: appId=cid-v1:1d21644b-7e61-4c5d-9e16-e5c56dba9811
vary: Origin
x-aspnet-version: 4.0.30319
x-powered-by: ASP.NET
{
    "collection": {
        "version": "1.0",
        "href": "https://conferenceapi.azurewebsites.net:443/speakers",
        "links": [],
        "items": [{
                "href": "https://conferenceapi.azurewebsites.net/speaker/1",
                "data": [{
                    "name": "Name",
   ...
````

Demo Conference API > Design > All operations > Outbound processing

````json
<policies>
    <inbound>
        <base />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <set-header name="X-Powered-By" exists-action="delete"/>
        <set-header name="X-AspNet-Version" exists-action="delete" />
        <redirect-content-urls />
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
````

After:

````json
transfer-encoding: chunked
vary: Origin
{
    "collection": {
        "version": "1.0",
        "href": "https://apim-learn-azure.azure-api.net/conference:443/speakers",
        "links": [],
        "items": [{
                "href": "https://apim-learn-azure.azure-api.net/conference/speaker/1",
                "data": [{
                   ...
````

#### Protect an API by adding rate limit policy (throttling)

Demo Conference API > All operations > Design > Inbound processing

````json
<policies>
    <inbound>
        <rate-limit-by-key calls="3" renewal-period="15" counter-key="@(context.Subscription.Id)" />
        <base />
    </inbound>
    ...
````

Test > GetSpeakers > Send x4 in less than 15 seconds

response "HTTP/1.1 429 Too Many Requests" on 4th call

## Azure Messaging Services

* Event Grid
* Event Hubs
* Service Bus

### Event

* An event is a lightweight notification of a condition or a state change.
* The publisher of the event has no expectation about how the event is handled.
* The consumer of the event decides what to do with the notification.
* Events can be discrete units or part of a series.

### Message

* A message is raw data produced by a service to be consumed or stored elsewhere.
* The message contains the data that triggered the message pipeline.
* The publisher of the message has an expectation about how the consumer handles the message.
* A contract exists between the two sides.

| Service | Purpose | Type | When to use |
|:---|:---:|:---:|:---:|:---:|
| Event Grid|  Reactive programming| Event distribution (discrete) | React to status changes |
| Event Hubs|  Big data pipeline | Event streaming (series) | Telemetry and distributed data streaming |
| Service Bus| High-value enterprise messaging | Message | Order processing and financial transactions |

## Develop Event-based Solutions

* Azure Event Grid
* Azure Notification Hubs
* Azure Event Hub

### Implement Solutions that use Azure Event Grid

Event Grid is an eventing backplane that enables event-driven, reactive programming.

It uses a publish-subscribe model. Publishers emit events, but have no expectation about which events are handled. Subscribers decide which events they want to handle.

Event Grid isn't a data pipeline, and doesn't deliver the actual object that was updated.

Event Grid supports dead-lettering for events that aren't delivered to an endpoint.

* dynamically scalable
* low cost
* serverless
* at least once delivery

#### Components

* Events - What happened.
* Event sources - Where the event took place.
   * Azure services
   * Cloud Events
   * Custom Events(anything)
* Topics - The endpoint where publishers send events.
* Event subscriptions - The endpoint or built-in mechanism to route events, sometimes to more than one handler. Subscriptions are also used by handlers to intelligently filter incoming events.
* Event handlers - The app or service reacting to the event.
   * Serverless (Functions)
   * Workflow & Integration (Logic Apps, Service Bus)
   * Buffering (Event Hub, Storage Queue)
   * Other (Webhooks, Automation)

Create a new Resource Group:

`az group create --name <resource-group-name> --location <location>`

Create an Event Grid Topic:

````xml
az eventgrid topic create --name <topic-name> \
  --location <location> \
  --resource-group <resource group name>
````

Make note of the endpoint value.

Get an access key:

`az eventgrid topic key list --name <topic-name> --resource-group <resource-group-name>`

Event Structure:

````json
[
  {
    "topic": string,
    "subject": string,
    "id": string,
    "eventType": string,
    "eventTime": string,
    "data":{
      object-unique-to-each-publisher
    }
  }
]
````

#### Handling Events with an Azure Function

* subscribe to events for recently added employees
* invoked only for employees that belong to the engineering department

````c#
...
 [FunctionName("newemployeehandler")]
      public static async Task<HttpResponseMessage> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post", Route = null)]
        HttpRequestMessage req,
        TraceWriter log)
         {
          log.Info("New Employee Handler Triggered");
          // Retrieve the contents of the request and
          // 
   ...
````

Event Grid will send to its subscribers two types of requests `SubscriptionValidation` and `Notification` that you can identify by inspecting a value from the header.

Create the event subscription

````xml
az eventgrid event-subscription create --name <event-subscription-name> \
  --resource-group <resource group name> \
  --topic-name <topic name> \
  --subject-ends-with engineering \
  --included-event-type employeeAdded \
  --endpoint <function endpoint>
````

#### Handling Events: Logic App and WebHook

* start with an Event Grid trigger
* resource type is `Microsoft.EventGrid.topics`
* filter for the event type with a `condition` action

Use a basic HTTP callback or WebHook to call a Web API

````c#
[Produces("application/json")]
  [Route("api/EmployeeUpdates")]
  public class EmployeeUpdatesController : Controller
  {
    private bool EventTypeSubcriptionValidation
      => HttpContext.Request.Headers["aeg-event-type"].FirstOrDefault() ==
        "SubscriptionValidation";
    private bool EventTypeNotification
      => HttpContext.Request.Headers["aeg-event-type"].FirstOrDefault() ==
        "Notification";
      ...
````

create the subscription

````xml
az eventgrid event-subscription create --name <event-subscription-name> \
  --resource-group <resource group name> \
  --topic-name <topic name> \
  --included-event-type employeeRemoved \
  --endpoint <function endpoint>
  ````

#### Route custom events to web endpoint with Event Grid

__Portal__

* Event Grid Topics > Add
* Create & Deploy your custom endpoint
* Event Grid Topic > Event Subscription > Create > Web Hook
   * point to custom endpoint

Send event to topic

__CLI__

Create a resource group

`az group create --name gridResourceGroup --location westus2`

Enable the Event Grid resource provider

`az provider register --namespace Microsoft.EventGrid`

Verify

`az provider show --namespace Microsoft.EventGrid --query "registrationState"`

Create a custom topic

````cmd
sitename=<your-site-name>

az group deployment create \
  --resource-group gridResourceGroup \
  --template-uri "https://raw.githubusercontent.com/Azure-Samples/azure-event-grid-viewer/master/azuredeploy.json" \
  --parameters siteName=$sitename hostingPlanName=viewerhost
````

Create a message endpoint

````cmd
sitename=<your-site-name>

az group deployment create \
  --resource-group gridResourceGroup \
  --template-uri "https://raw.githubusercontent.com/Azure-Samples/azure-event-grid-viewer/master/azuredeploy.json" \
  --parameters siteName=$sitename hostingPlanName=viewerhost
  ````

  Subscribe to a custom topic

  ````cmd
  endpoint=https://$sitename.azurewebsites.net/api/updates

az eventgrid event-subscription create \
  --source-resource-id "/subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.EventGrid/topics/$topicname" \
  --name demoViewerSub \
  --endpoint $endpoint
  ````

  Send an event to your custom topic

  ````cmd
  endpoint=$(az eventgrid topic show --name $topicname -g gridResourceGroup --query "endpoint" --output tsv)
key=$(az eventgrid topic key list --name $topicname -g gridResourceGroup --query "key1" --output tsv)

event='[ {"id": "'"$RANDOM"'", "eventType": "recordInserted", "subject": "myapp/vehicles/motorcycles", "eventTime": "'`date +%Y-%m-%dT%H:%M:%S%z`'", "data":{ "make": "Ducati", "model": "Monster"},"dataVersion": "1.0"} ]'

curl -X POST -H "aeg-sas-key: $key" -d "$event" $endpoint
````

__PowerShell__

Create a resource group

`New-AzResourceGroup -Name gridResourceGroup -Location westus2`

Enable Event Grid resource provider

`Register-AzResourceProvider -ProviderNamespace Microsoft.EventGrid`

Verify

`Get-AzResourceProvider -ProviderNamespace Microsoft.EventGrid`

When `RegistrationStatus` is `Registered`, you're ready to continue.

Create a custom topic

````ps
$topicname="<your-topic-name>"

New-AzEventGridTopic -ResourceGroupName gridResourceGroup -Location westus2 -Name $topicname
````

Create a message endpoint

````ps
$sitename="<your-site-name>"

New-AzResourceGroupDeployment `
  -ResourceGroupName gridResourceGroup `
  -TemplateUri "https://raw.githubusercontent.com/Azure-Samples/azure-event-grid-viewer/master/azuredeploy.json" `
  -siteName $sitename `
  -hostingPlanName viewerhost
  ````

  Subscribe to a topic

  ````ps
  $endpoint="https://$sitename.azurewebsites.net/api/updates"

New-AzEventGridSubscription `
  -EventSubscriptionName demoViewerSub `
  -Endpoint $endpoint `
  -ResourceGroupName gridResourceGroup `
  -TopicName $topicname
  ````
Send an event to your topic

````ps
$endpoint = (Get-AzEventGridTopic -ResourceGroupName gridResourceGroup -Name $topicname).Endpoint
$keys = Get-AzEventGridTopicKey -ResourceGroupName gridResourceGroup -Name $topicname

$eventID = Get-Random 99999

#Date format should be SortableDateTimePattern (ISO 8601)
$eventDate = Get-Date -Format s

#Construct body using Hashtable
$htbody = @{
    id= $eventID
    eventType="recordInserted"
    subject="myapp/vehicles/motorcycles"
    eventTime= $eventDate   
    data= @{
        make="Ducati"
        model="Monster"
    }
    dataVersion="1.0"
}

#Use ConvertTo-Json to convert event body from Hashtable to JSON Object
#Append square brackets to the converted JSON payload since they are expected in the event's JSON payload syntax
$body = "["+(ConvertTo-Json $htbody)+"]"

Invoke-WebRequest -Uri $endpoint -Method POST -Body $body -Headers @{"aeg-sas-key" = $keys.Key1}
````

### Implement Solutions that use Azure Notification Hubs

Azure Notification Hubs provide an easy-to-use and scaled-out push engine (targets mobile)

Push notifications are delivered through platform-specific infrastructures called Platform Notification Systems (PNSes).

* Cross platforms: common interface to push to all platforms, Device handle management, Support for all major push platforms
* Cross backends: Cloud or on-premises, .NET, Node.js, Java, Python, etc.
* Rich set of delivery patterns: Broadcast, Push to [device|user|segment], Silent (push to pull), etc.
* Rich telemetry:General push, device, error, and operation telemetry and Per-message telemetry
* Scalability
* Security: Shared Access Secret (SAS) or federated authentication

#### Create an Azure Notification Hub

__Portal__

All services > Mobile > Notification Hub > Create

Create a new namespace or select an existing namespace

Add name for notification hub

Create

After deployment go to the resource

Access Policies > Note that the two connection strings

__CLI__

Create a resource group

`az group create --name spnhubrg --location eastus`

Check for Notification Hubs namepsace availability

`az notification-hub namespace check-availability --name spnhubns`

````json
{
"id": "/subscriptions/yourSubscriptionID/providers/Microsoft.NotificationHubs/checkNamespaceAvailability",
"isAvailable": true,
"location": null,
"name": "spnhubns",
"properties": false,
"sku": null,
"tags": null,
"type": "Microsoft.NotificationHubs/namespaces/checkNamespaceAvailability"
}
````

Create a Notification Hubs namespace

`az notification-hub namespace create --resource-group spnhubrg --name spnhubns  --location eastus --sku Free`

Get a list of namespaces

`az notification-hub namespace list --resource-group spnhubrg`

Create notification hubs

`az notification-hub create --resource-group spnhubrg --namespace-name spnhubns --name spfcmtutorial1nhub --location eastus --sku Free`

Get a list of notification hubs

`az notification-hub list --resource-group spnhubrg --namespace-name spnhubns --output table`

List access policies

`az notification-hub authorization-rule list --resource-group spnhubrg --namespace-name spnhubns --notification-hub-name spfcmtutorial1nhub --output table`

Create and customize your own access policy

`az notification-hub authorization-rule create --resource-group spnhubrg --namespace-name spnhubns --notification-hub-name spfcmtutorial1nhub --name spnhub1key --rights Listen Manage Send`

#### Set up push notifications in a notification hub

Notification Hub > Apple (APNS), Google (GCM/FCM), etc.

Authentication Mode: Certificate or Token

### Implement Solutions that use Azure Event Hub

Azure Event Hubs is a big data pipeline. It facilitates the capture, retention, and replay of telemetry and event stream data.

The data can come from many concurrent sources.

* low latency
* capable of receiving and processing millions of events per second
* at least once delivery

Architecture Components

* Event producers: Any entity that sends data to an event hub. Event publishers can publish events using HTTPS or AMQP 1.0 or Apache Kafka (1.0 and above)
* Partitions: Each consumer only reads a specific subset, or partition, of the message stream.
* Consumer groups: A view (state, position, or offset) of an entire event hub. Consumer groups enable consuming applications to each have a separate view of the event stream. They read the stream independently at their own pace and with their own offsets.
* Throughput units: Pre-purchased units of capacity that control the throughput capacity of Event Hubs.
* Event receivers: Any entity that reads event data from an event hub. All Event Hubs consumers connect via the AMQP 1.0 session. The Event Hubs service delivers events through a session as they become available. All Kafka consumers connect via the Kafka protocol 1.0 and later.

#### Terminology

* Namespace: unique scoping container.
* Event Hubs for Apache Kafka: endpoint that enables customers to talk to Event Hubs using the Kafka protocol.
* Event publishers: Any entity that sends data to an event hub is an event producer, or event publisher.
* Publishing an event: publish an event via AMQP 1.0, Kafka 1.0 (and later), or HTTPS.
* Publisher policy: Publisher policies are run-time features designed to facilitate large numbers of independent event publishers.
* Capture: capture the streaming data in Event Hubs and save it to your choice of either a Blob storage account, or an Azure Data Lake Service account.
* Partitions: Event Hubs provides message streaming through a partitioned consumer pattern in which each consumer only reads a specific subset, or partition, of the message stream. This pattern enables horizontal scale for event processing and provides other stream-focused features that are unavailable in queues and topics.
* SAS tokens: Event Hubs uses Shared Access Signatures, which are available at the namespace and event hub level.
* Event consumers: Any entity that reads event data from an event hub is an event consumer.
* Consumer groups: A consumer group is a view (state, position, or offset) of an entire event hub that enables the publish/subscribe mechanism.

#### Create an event hub

__Portal__

Resource Groups > Add

All services > Event Hubs > Create namespace

Event Hubs Namespace > Add Event Hub

* Partition Count
* Message Retention

__CLI__

Sign in to Azure

`az login`

`az account set --subscription MyAzureSub`

Create a resource group

`az group create --name <resource group name> --location eastus`

Create an Event Hubs namespace

`az eventhubs namespace create --name <Event Hubs namespace> --resource-group <resource group name> -l <region, for example: East US>`

Create an event hub

`az eventhubs eventhub create --name <event hub name> --resource-group <resource group name> --namespace-name <Event Hubs namespace>`

Options

`--message-retention` is the number of days to retain messages (1-7) and depends on Namespace sku. if Namespace sku is Basic than value should be one and is Manadatory parameter.

`--partition-count` Number of partitions created for the Event Hub. By default, allowed values are 2-32. Lower value of 1 is supported with Kafka enabled namespaces.

__PowerShell__

Create a resource group

`New-AzResourceGroup –Name myResourceGroup –Location eastus`

Create an Event Hubs namespace

`New-AzEventHubNamespace -ResourceGroupName myResourceGroup -NamespaceName namespace_name -Location eastus`

Create an event hub

`New-AzEventHub -ResourceGroupName myResourceGroup -NamespaceName namespace_name -EventHubName eventhub_name -MessageRetentionInDays 3`

__ARM template__

An ARM template is a JavaScript Object Notation (JSON) file that defines the infrastructure and configuration for your project.

Pseudo-template (object properties ommitted):

````json
{
   parameters: {
      projectName, location, eventHubSku
   },
   variables: {
      eventHubNamespaceName, eventHubName
   },
   resources : {
      Microsoft.EventHub/namespaces, Microsoft.EventHub/namespaces/eventhubs
   }
}
````

````ps
$projectName = Read-Host -Prompt "Enter a project name that is used for generating resource names"
$location = Read-Host -Prompt "Enter the location (i.e. centralus)"
$resourceGroupName = "${projectName}rg"
$templateUri = "https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/101-eventhubs-create-namespace-and-eventhub/azuredeploy.json"

New-AzResourceGroup -Name $resourceGroupName -Location $location
New-AzResourceGroupDeployment -ResourceGroupName $resourceGroupName -TemplateUri $templateUri -projectName $projectName

Write-Host "Press [ENTER] to continue ..."
````

## Develop message-based solutions

* Azure Service Bus
* Azure Queues

### Technology Selection

Storage queues

* store over 80 GB of messages in a queue.
* track progress for processing a message inside of the queue. This is useful if the worker processing a message crashes. A subsequent worker can then use that information to continue from where the prior worker left off.
* server side logs of all of the transactions executed against your queues.

Service Bus queues

* receive messages without having to poll the queue. (long-polling with supported TCP-based protocols)
* Your solution requires the queue to provide a guaranteed first-in-first-out (FIFO) ordered delivery.
* automatic duplicate detection.
* process messages as parallel long-running streams
* transactional behavior and atomicity when sending or receiving multiple messages from a queue.
* messages that can exceed 64 KB but will not likely approach the 256 KB limit
* role-based access model to the queues, and different rights/permissions for senders and receivers
* queue size will not grow larger than 80 GB.
* AMQP 1.0 standards-based messaging protocol.
* planning on an eventual migration from queue-based point-to-point communication to a message exchange pattern
* support the "At-Most-Once" delivery guarantee
* publish and consume batches of messages.

### Implement Solutions that use Azure Service Bus

* managed enterprise integration message broker

Service Bus is intended for traditional enterprise applications.

* Transactions
* ordering
* duplicate detection
* instantaneous consistency

When handling high-value messages that cannot be lost or duplicated, use Azure Service Bus.

Secure communication across hybrid cloud solutions and can connect existing on-premises systems to cloud solutions.

* reliable asynchronous message delivery (enterprise messaging as a service) that requires polling
* advanced messaging features like FIFO, batching/sessions, transactions, dead-lettering, temporal control, routing and filtering, and duplicate detection
* at least once delivery
* optional in-order delivery

* Namespaces: container for all messaging components. Multiple queues and topics can be in a single namespace, and namespaces often serve as application containers.
* Queues: Messages are sent to and received from queues.
* Topics: publish/subscribe scenarios.

Service Bus queues support a brokered messaging communication model. When using queues, components of a distributed application do not communicate directly with each other; instead they exchange messages via a queue, which acts as an intermediary (broker)

#### Azure Portal

Integration > Service Bus

Get connection string

Service bus > Shared access policies > RootManageSharedAccessKey

Create a queue in the Azure portal

Service Bus Namespace > Queues > Add > ... > Create

#### CLI

Create a resource group

`az group create --name az204servicebus --location westus2`

Create service bus namespace

`az servicebus namespace create --resource-group az204servicebus --name az204learnservicebuscli --location westus2`

Create a queue

`az servicebus queue create --resource-group az204servicebus --namespace-name az204learnservicebuscli --name queuecli`

#### PowerShell

Create a resource group

`New-AzResourceGroup –Name ContosoRG –Location eastus`

Create service bus namespace

`New-AzServiceBusNamespace -ResourceGroupName ContosoRG -Name ContosoSBusNS -Location eastus`

Create a queue

`New-AzServiceBusQueue -ResourceGroupName ContosoRG -NamespaceName ContosoSBusNS -Name ContosoOrdersQueue`

Get connection string

`Get-AzServiceBusKey -ResourceGroupName ContosoRG -Namespace ContosoSBusNS -Name RootManageSharedAccessKey`

Send messages to the queue

````c#
queueClient = new QueueClient(ServiceBusConnectionString, QueueName);

// Create a new message to send to the queue.
string messageBody = $"Message {i}";
var message = new Message(Encoding.UTF8.GetBytes(messageBody));

// Send the message to the queue.
await queueClient.SendAsync(message);
````

Receive messages from the queue

````c#
  // Configure the message handler options in terms of exception handling, number of concurrent messages to deliver, etc.
  var messageHandlerOptions = new MessageHandlerOptions(ExceptionReceivedHandler)
  {
      // Maximum number of concurrent calls to the callback ProcessMessagesAsync(), set to 1 for simplicity.
      // Set it according to how many messages the application wants to process in parallel.
      MaxConcurrentCalls = 1,

      // Indicates whether the message pump should automatically complete the messages after returning from user callback.
      // False below indicates the complete operation is handled by the user callback as in ProcessMessagesAsync().
      AutoComplete = false
  };

  // Register the function that processes messages.
  queueClient.RegisterMessageHandler(ProcessMessagesAsync, messageHandlerOptions);
````

#### Service Bus Topics and Subscriptions

#### Portal

Service Bus Namespace > Topics > Add > ... > Create

Topic > Subscriptions > Add > ... > Create

#### CLI

Create a topic

`az servicebus topic create --resource-group az204servicebus   --namespace-name az204learnservicebuscli --name topic2`

Create subscriptions

`az servicebus topic subscription create --resource-group az204servicebus --namespace-name az204learnservicebuscli --topic-name topic2 --name sub2a`

`az servicebus topic subscription create --resource-group az204servicebus --namespace-name az204learnservicebuscli --topic-name topic2 --name sub2b`

Create a filter on subscription 2a

`az servicebus topic subscription rule create --resource-group az204servicebus --namespace-name az204learnservicebuscli --topic-name topic2 --subscription-name sub2a --name MyFilter --filter-sql-expression "StoreId IN ('Store1','Store2','Store3')"`

Send messages to the topic

````c#
topicClient = new TopicClient(ServiceBusConnectionString, TopicName);

// Create a new message to send to the topic.
string messageBody = $"Message {i}";
var message = new Message(Encoding.UTF8.GetBytes(messageBody));

// Write the body of the message to the console.
Console.WriteLine($"Sending message: {messageBody}");

// Send the message to the topic.
await topicClient.SendAsync(message);
````

Receive messages from the subscription

````c#
subscriptionClient = new SubscriptionClient(ServiceBusConnectionString, TopicName, SubscriptionName);

// Configure the message handler options in terms of exception handling, number of concurrent messages to deliver, etc.
var messageHandlerOptions = new MessageHandlerOptions(ExceptionReceivedHandler)
{
    // Maximum number of concurrent calls to the callback ProcessMessagesAsync(), set to 1 for simplicity.
    // Set it according to how many messages the application wants to process in parallel.
    MaxConcurrentCalls = 1,

    // Indicates whether the message pump should automatically complete the messages after returning from user callback.
    // False below indicates the complete operation is handled by the user callback as in ProcessMessagesAsync().
    AutoComplete = false
};

// Register the function that processes messages.
subscriptionClient.RegisterMessageHandler(ProcessMessagesAsync, messageHandlerOptions);

static async Task ProcessMessagesAsync(Message message, CancellationToken token)
{
  // Process the message.
  Console.WriteLine($"Received message: SequenceNumber:{message.SystemProperties.SequenceNumber} Body:{Encoding.UTF8.GetString(message.Body)}");
  ...
````

### Implement Solutions that use Azure Queue Storage Queues

* Service for storing large numbers of messages.
* Access messages with authenticated calls using HTTP or HTTPS.
* A queue message can be up to 64 KB in size.
* A queue's total capacity limit of a storage account.
* Use Case: create a backlog of work to process asynchronously.

Concepts:

* Storage account: access point
* Queue: contains a set of messages
* Message: any format, of up to 64 KB.
   * pre v2017-07-29 max TTL is 7 days
   * v2017-07-29 or later max TTL can be any positive number, or -1 indicating that the message doesn't expire
   * default: 7 days

#### Create a queue and add a message with the Azure portal

Create a queue

Storage Account > Queue service > Queues Add

* Add a message: A message can be up to 64 KB in size.
* View message properties
* Dequeue a message

Retrieve connection string and set a Environment Variable

`setx AZURE_STORAGE_CONNECTION_STRING "<yourconnectionstring>`

````c#
// Get the connection string from app settings
string connectionString = ConfigurationManager.AppSettings["StorageConnectionString"];

// Instantiate a QueueClient which will be used to create and manipulate the queue
QueueClient queueClient = new QueueClient(connectionString, queueName);

await theQueue.CreateIfNotExistsAsync();

// insert never expiring message
await theQueue.SendMessageAsync(newMessage, default, TimeSpan.FromSeconds(-1), default);

// get queue length
QueueProperties properties = queueClient.GetProperties();
// Retrieve the cached approximate message count.
int cachedMessagesCount = properties.ApproximateMessagesCount;

// Peek at the next message
PeekedMessage[] peekedMessage = queueClient.PeekMessages();

// dequeue
QueueMessage[] retrievedMessage = await theQueue.ReceiveMessagesAsync(1);
string theMessage = retrievedMessage[0].MessageText;
await theQueue.DeleteMessageAsync(retrievedMessage[0].MessageId, retrievedMessage[0].PopReceipt);
````
