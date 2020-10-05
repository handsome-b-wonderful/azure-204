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

Azure Logic Apps provides a prebuilt logic app Azure Resource Manager template that you can reuse for creating logic apps and to define the resources and parameters to use for deployment

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

template parameters for sensitive values usees the securestring or secureobject parameter types to reference the key, with the related value then stored in Azure Key Vault.

## Implement API management


### create an APIM instance


### configure authentication for APIs


### define policies for APIs




## Develop event-based solutions

### implement solutions that use Azure Event Grid

### implement solutions that use Azure Notification Hubs

### implement solutions that use Azure Event Hub



## Develop message-based solutions

### implement solutions that use Azure Service Bus

### implement solutions that use Azure Queue Storage queues
