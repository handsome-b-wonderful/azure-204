# Implement Azure Security (15-20%)

Azure Active Directory (Azure AD) is a centralized identity provider in the cloud.

* Use of multi-factor authentication
* Single Sign On (SSO)

Microsoft identity platform simplifies authorization and authentication for application developers by providing identity as a service.

## Implement User Authentication and Authorization

__Authentication__ is the process of proving you are who you say you are.

__Authorization__ is the act of granting an authenticated party permission to do something.

* OAuth vs OpenID Connect: OAuth is used for authorization and OpenID Connect (OIDC) is used for authentication. OpenID Connect is built on top of OAuth 2.0.
* OAuth vs SAML: OAuth is used for authorization and SAML is used for authentication.
* OpenID Connect vs SAML: Both OpenID Connect and SAML are used to authenticate a user and are used to enable Single Sign On. SAML authentication frequently used in enterprise applications; OpenID Connect is commonly used for cloud-based apps.

### Implement OAuth2 Authentication

Requires:

* An API Management instance
* An API being published that uses the API Management instance
* An Azure AD tenant

Create a resource group

`az group create --name az204-learn-oauth --location canadacentral`

Create an App Plan

`az appservice plan create --name az204learnoauthappplan --resource-group az204-learn-oauth --sku F1 --is-linux`

Create a (service) web API

`az --% webapp create --resource-group az204-learn-oauth --plan az204learnoauthappplan --name webapi-oauth-service --runtime "DOTNETCORE|3.1"`

Create an API amanagement service

`az apim create --name myapim --resource-group az204-learn-oauth --publisher-name Contoso --publisher-email admin@contoso.com --no-wait`

Import the backend API into API Management (as per "Implement API management" scenarios)

#### Register an application in Azure AD to represent the API

To protect an API with Azure AD, first register an application in Azure AD that represents the API.

Search for "App registrations" > New registration

* Name: meaningful name
* Supported account types: scenario-specific
* Redirect URI: blank

Complete the registration and record the "Application (client) ID" from the overview page

Select `Expose an API` and set the "Application ID URI" with the default value.

Select the `Add` a scope button to display the Add a scope page. Then create a new scope that's supported by the API (ex: `Files.Read`)

Add any other scopes, as required, and record them.

#### Register another application in Azure AD to represent a client application

Every client application that calls the API needs to be registered as an application in Azure AD.

App registrations > New registration

* Name: meaningful name
* Supported account types: Accounts in any organizational directory (Any Azure AD directory - Multitenant)
* Redirect URI: select `web` and leave blank

Complete the registration and record the "Application (client) ID" from the overview page.

Create a client secret

Certificates & secrets > New client secret

Add a client secret > Description, when the key should expire > Add

Record the secret `key` value.

#### Grant permissions in Azure AD

App registrations > client app > API permissions > Add permissions

Select an API > My APIs > backend app (API)

Delegated Permissions > desired permissions > Add permissions

OPTIONAL: API permissions > Grant admin consent for your-tennant-name-here to grant consent on behalf of all users in this directory.

#### Enable OAuth 2.0 user authorization in the Developer Console

* created your applications in Azure AD
* granted proper permissions to allow the client-app to call the backend-app.

API Management instance > OAuth 2.0 > Add

* Display name: meaningful name
* Description: optional
* Client registration page URL: placeholder value `http://localhost`
    * points to a page that users can use to create and configure their own accounts for OAuth 2.0 providers that support this. (not implemented here)
* Authorization grant types: Authorization code
* Authorization endpoint URL: App registrations > Endpoint > OAuth 2.0 Authorization Endpoint
    * Authorization request method: Post
* Token endpoint URL: App registrations > Endpoint > OAuth 2.0 Token Endpoint
    * use `v2`
* Default scope: scope created above
* Client ID: Application ID
* Client secret: secret key

Create

Azure Active Directory > client registration > Authentication

Platform configurations > Add a platform

* Type: Web
* Redirect URI: redirect_url

Configure

Now that you have configured an OAuth 2.0 authorization server, the Developer Console can obtain access tokens from Azure AD.

Enable OAuth 2.0 user authorization for your API

API Management > APIs > my-api

Settings > Security > OAuth 2.0 > configured OAuth 2.0 server

Save

#### Configure a JWT validation policy to pre-authorize requests

Add a Validate JWT policy to pre-authorize requests in API Management to prevent them from just being passed to the back-end by the API management

If a request does not have a valid token, API Management blocks it.

Add inbound policy

````xml
<validate-jwt header-name="Authorization" failed-validation-httpcode="401" failed-validation-error-message="Unauthorized. Access token is missing or invalid.">
    <openid-config url="https://login.microsoftonline.com/common/v2.0/.well-known/openid-configuration" />
    <required-claims>
        <claim name="aud">
            <value>{Application ID of backend-app}</value>
        </claim>
    </required-claims>
</validate-jwt>
````

### Create and Implement Shared Access Signatures

Types of shared access signatures

* User delegation SAS: A user delegation SAS is secured with Azure Active Directory (Azure AD) credentials and also by the permissions specified for the SAS. A user delegation SAS applies to Blob storage only.

* Service SAS: A service SAS is secured with the storage account key. A service SAS delegates access to a resource in only one of the Azure Storage services: Blob storage, Queue storage, Table storage, or Azure Files.

* Account SAS: An account SAS is secured with the storage account key. An account SAS delegates access to resources in one or more of the storage services. All of the operations available via a service or user delegation SAS are also available via an account SAS.

A shared access signature is a signed URI that points to one or more storage resources and includes a token that contains a special set of query parameters. The token indicates how the resources may be accessed by the client. One of the query parameters, the signature, is constructed from the SAS parameters and signed with the key that was used to create the SAS. This signature is used by Azure Storage to authorize access to the storage resource.

Sign a SAS token:

1. With a user `delegation key` that was created using Azure Active Directory (Azure AD) credentials. A user delegation SAS is signed with the user delegation key.

2. With the `storage account key` (Shared Key). Both a service SAS and an account SAS are signed with the storage account key. To create a SAS that is signed with the account key, an application must have access to the account key.

If a SAS is leaked, it can be used by anyone who obtains it.

If a SAS provided to a client application expires and the application is unable to retrieve a new SAS from your service, then the application's functionality may be hindered.

* Always use HTTPS
* Have a revocation plan in place for a SAS
* Define a stored access policy for a service SAS. Stored access policies give you the option to revoke permissions for a service SAS without having to regenerate the storage account keys. Set the expiration on these very far in the future (or infinite) and make sure it's regularly updated to move it farther into the future.
* Use near-term expiration times on an ad hoc SAS service SAS or account SAS.
* Have clients automatically renew the SAS if necessary. 
* Be careful with SAS start time. If you set the start time for a SAS to the current time, you may observe failures occurring intermittently for the first few minutes due to different machines having slightly different current times (known as clock skew)
* Be specific with the resource to be accessed.
* Understand that your account will be billed for any usage, including via a SAS.
* Use Azure Monitor and Azure Storage logs to monitor your application.

#### two forms of Shared Access Signatures:

* Ad hoc: The start time, expiry time, and permissions for the SAS are all specified on the SAS URI.
* Stored access policy: A stored access policy is defined on a resource container, such as a blob container. A policy can be used to manage constraints for one or more shared access signatures. When you associate a SAS with a stored access policy, the SAS inherits the constraints - the start time, expiry time, and permissions - defined for the stored access policy.

SAS is valid until:

1. The expiry time specified on the SAS is reached.
2. The expiry time specified on the stored access policy referenced by the SAS is reached.
    * The time interval has elapsed.
    * The stored access policy is modified to have an expiry time in the past. Changing the expiry time is one way to revoke the SAS.
3. The stored access policy referenced by the SAS is deleted, which is another way to revoke the SAS.
4. The account key that was used to create the SAS is regenerated. Regenerating the key causes all applications that use the previous key to fail authentication. Update all components to the new key.

#### Create a stored policy and SAS

PowerShell

````ps
# Get the access key for the Azure Storage account
$storageAccountKey = (Get-AzStorageAccountKey `
    -ResourceGroupName $resourceGroupName `
    -Name $storageAccountName)[0].Value

# Create an Azure Storage context
$storageContext = New-AzStorageContext `
    -StorageAccountName $storageAccountName `
    -StorageAccountKey $storageAccountKey

# Create a stored access policy for the Azure storage container
New-AzStorageContainerStoredAccessPolicy `
   -Container $containerName `
   -Policy $policy `
   -Permission "rl" `
   -ExpiryTime "12/31/2025 08:00:00" `
   -Context $storageContext

# Get the stored access policy or policies for the Azure storage container
Get-AzStorageContainerStoredAccessPolicy `
    -Container $containerName `
    -Context $storageContext

# Generates an SAS token for the Azure storage container
New-AzStorageContainerSASToken `
    -Name $containerName `
    -Policy $policy `
    -Context $storageContext

<# Removes a stored access policy from the Azure storage container
Remove-AzStorageContainerStoredAccessPolicy `
    -Container $containerName `
    -Policy $policy `
    -Context $storageContext
#>

# upload a file for a later example
Set-AzStorageblobcontent `
    -File "./sampledata/sample.log" `
    -Container $containerName `
    -Blob "samplePS.log" `
    -Context $storageContext
````

CLI

````
# set variables
set AZURE_STORAGE_ACCOUNT=STORAGEACCOUNT
set AZURE_STORAGE_CONTAINER=STORAGECONTAINER

#Login
az login

# If you have multiple subscriptions, set the one to use
# az account set --subscription SUBSCRIPTION

# Retrieve the primary key for the storage account
az storage account keys list --account-name %AZURE_STORAGE_ACCOUNT% --query "[0].{PrimaryKey:value}" --output table

#set variable for primary key
set AZURE_STORAGE_KEY=PRIMARYKEY

# Create stored access policy on the containing object
az storage container policy create --container-name %AZURE_STORAGE_CONTAINER% --name myPolicyCLI --account-key %AZURE_STORAGE_KEY% --account-name %AZURE_STORAGE_ACCOUNT% --expiry 2025-12-31 --permissions rl

# List stored access policies on a containing object
az storage container policy list --container-name %AZURE_STORAGE_CONTAINER% --account-key %AZURE_STORAGE_KEY% --account-name %AZURE_STORAGE_ACCOUNT%

# Generate a shared access signature for the container
az storage container generate-sas --name myPolicyCLI --account-key %AZURE_STORAGE_KEY% --account-name %AZURE_STORAGE_ACCOUNT%

# Reversal
# az storage container policy delete --container-name %AZURE_STORAGE_CONTAINER% --name myPolicyCLI --account-key %AZURE_STORAGE_KEY% --account-name %AZURE_STORAGE_ACCOUNT%

# upload a file for a later example
az storage blob upload --container-name %AZURE_STORAGE_CONTAINER% --account-key %AZURE_STORAGE_KEY% --account-name %AZURE_STORAGE_ACCOUNT% --name sampleCLI.log --file "./sampledata/sample.log"
````

### Register Apps and use Azure Active Directory to Authenticate Users

Steps:

* Configure access rules: Configure per-application access rules. Examples: require MFA or only allow access to users on trusted networks.
* Configure the app to require user assignment and assign users
* Suppress user consent: For applications that you trust, you can simplify the user experience by consenting to the application on behalf of your organization.

#### Register an application

see also "Implement OAuth2 Authentication" above.

Azure Active Directory > Manage > App registrations > New registration

Specify who can use the application, sometimes referred to as the sign-in audience.

* Accounts in this organizational directory only
* Accounts in any organizational directory
* Accounts in any organizational directory and personal Microsoft accounts
* Personal Microsoft accounts

The `Application (client) ID` or `client ID` uniquely identifies the application in the Microsoft identity platform.

Add a redirect URI

A redirect URI is the location where the Microsoft identity platform redirects a user's client and sends security tokens after authentication.

Configure platform settings

App registrations > Manage > Authentication > Platform configurations > Add platform

* Web
* Single-page application
* iOS / macOS
* Android
* Mobile and desktop applications

Add credentials

Credentials are used by confidential client applications that access a web API. Examples of confidential clients are web apps, other web APIs, or service- and daemon-type applications. Credentials allow your application to authenticate as itself, requiring no interaction from a user at runtime.

Add a certificate

Sometimes called a `public key`, certificates are the recommended credential type as they provide a higher level of assurance than a client secret.

App registrations > Certificates & secrets > Upload certificate

Add a client secret

The client secret, known also as an `application password`, is a string value your app can use in place of a certificate to identity itself.

App registrations > Certificates & secrets > New client secret

### Control Access to Resources by using Role-Based Access Controls (RBAC)

Azure RBAC helps you manage who has access to Azure resources, what they can do with those resources, and what areas they have access to.

#### Security principal

A `security principal` is an object that represents a user, group, service principal, or managed identity that is requesting access to Azure resources. You can assign a role to any of these security principals.

#### Role definition

A `role definition` is a collection of permissions. It's typically just called a `role`. A role definition lists the operations that can be performed, such as read, write, and delete. Roles can be high-level, like owner, or specific, like virtual machine reader.

#### Scope

`Scope` is the set of resources that the access applies to. When you assign a role, you can further limit the actions allowed by defining a scope.

Specify a scope at four levels:

1. management group
2. subscription
3. resource group
4. resource

#### Role assignments

A `role assignment` is the process of attaching a role definition to a user, group, service principal, or managed identity at a particular scope for the purpose of granting access. Access is granted by creating a role assignment, and access is revoked by removing a role assignment.

Azure RBAC is an additive model, so your effective permissions are the sum of your role assignments

 Azure RBAC supports deny assignments in a limited way. Deny assignments block users from performing specified actions even if a role assignment grants them access. Deny assignments take precedence over role assignments.

#### Four Fundamental Azure Roles

1. Owner: Full access to all resources, Delegate access to others
2. Contributor: create and manage resource, create AD tenant, NO GRANT (all resource types)
3. Reader: View Azure resources (all resource types)
4. User Access Administrator: Manage user access to Azure resources

At a high level, Azure roles control permissions to manage Azure resources, while Azure AD roles control permissions to manage Azure Active Directory resources.

#### CLI Operations

List all roles

`az role definition list`

List a role definition

`az role definition list --name "Contributor"`

List permissions of a role definition

`az role definition list --name "Contributor" --output json --query '[].{actions:permissions[0].actions, notActions:permissions[0].notActions}'`

Add a role

Step 1: Determine who needs access

Step 2: Find the appropriate role

Step 3: Identify the needed scope

Step 4: Add role assignment

resource scope:

`az role assignment create --assignee "{assignee}" \
--role "{roleNameOrId}" \
--scope "/subscriptions/{subscriptionId}/resourcegroups/{resourceGroupName}/providers/{providerName}/{resourceType}/{resourceSubType}/{resourceName}"`

Subscription scope:

`az role assignment create --assignee "{assignee}" \
--role "{roleNameOrId}" \
--subscription "{subscriptionNameOrId}"`

Remove role assignment

`az role assignment delete --assignee "patlong@contoso.com" \
--role "Virtual Machine Contributor" \
--resource-group "pharma-sales"`

## Implement Secure Cloud Solutions

### Secure App Configuration Data by using the App Configuration and KeyVault API

Azure App Configuration provides a service to centrally manage application settings and feature flags

App Configuration is secure but not as many features as Key Vault which provides hardware-level encryption, granular access policies, and management operations such as certificate rotation.

Azure App Service allows you to define app settings for each App Service instance. These settings are passed as environment variables to the application code. You can associate a setting with a specific deployment slot/

Azure App Configuration allows you to define settings that can be shared among multiple apps. This includes apps running in App Service, as well as other platforms. Your application code accesses these settings through the configuration providers for .NET and Java, through the Azure SDK, or directly via REST APIs.

#### CLI

Create an Azure App Configuration Store

````
appConfigName=myTestAppConfigStore
#resource name must be lowercase
myAppConfigStoreName=${appConfigName,,}
myResourceGroupName=$appConfigName"Group"

# Create resource group 
az group create --name $myResourceGroupName --location eastus

# Create the Azure AppConfig Service resource and query the hostName
appConfigHostname=$(az appconfig create \
  --name $myAppConfigStoreName \
  --location eastus \
  --resource-group $myResourceGroupName \
  --query endpoint \
  --sku free \
  -o tsv
  )

# Get the AppConfig connection string 
appConfigConnectionString=$(az appconfig credential list \
--resource-group $myResourceGroupName \
--name $myAppConfigStoreName \
--query "[?name=='Secondary Read Only'] .connectionString" -o tsv)

# Echo the connection string for use in your application
echo "$appConfigConnectionString"
````

Work with key-values

````
appConfigName=myTestAppConfigStore
newKey="TestKey"
refKey="KeyVaultReferenceTestKey"
uri="[URL to value stored in Key Vault]"
uri2="[URL to another value stored in Key Vault]"

# Create a new key-value 
az appconfig kv set --name $appConfigName --key $newKey --value "Value 1"

# List current key-values
az appconfig kv list --name $appConfigName

# Update new key's value
az appconfig kv set --name $appConfigName --key $newKey --value "Value 2"

# List current key-values
az appconfig kv list --name $appConfigName
````

Import

`az appconfig kv import --name myTestAppConfigStore --source file --path ~/Import.json`

Export

`az appconfig kv export --name myTestAppConfigStore --file ~/Export.json`

#### Portal

Create a resource > App Configuration > Create

Create an ASP.NET Core web app

`dotnet new mvc --no-https --output TestAppConfig`

Add Secret Manager

`dotnet user-secrets init`

A `UserSecretsId` element containing a GUID is added to the `.csproj` file:

````xml
<Project Sdk="Microsoft.NET.Sdk.Web">
    <PropertyGroup>
        <TargetFramework>netcoreapp3.1</TargetFramework>
        <UserSecretsId>79a3edd0-2092-40a2-a04d-dcb46d5ca9ed</UserSecretsId>
    </PropertyGroup>
</Project>
````

Connect to the App Configuration store

`dotnet add package Microsoft.Azure.AppConfiguration.AspNetCore`

Add a secret

````
> dotnet user-secrets set ConnectionStrings:AppConfig "<your_connection_string>"

Successfully saved ConnectionStrings:AppConfig = <your_connection_string> to the secret store.
````

In `Program.cs`, add a reference to the .NET Core Configuration API namespace.

`using Microsoft.Extensions.Configuration;`

Update the `CreateHostBuilder` method to use App Configuration by calling the `AddAzureAppConfiguration` method.

````c#
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
            webBuilder.ConfigureAppConfiguration(config =>
            {
                var settings = config.Build();
                var connection = settings.GetConnectionString("AppConfig");
                config.AddAzureAppConfiguration(connection);
            }).UseStartup<Startup>());
````

Add key-values to the App Configuration store

* Key= TestApp:Settings:BackgroundColor, Value= #f30
* Key= TestApp:Settings:FontColor, Value= #00f
* Key= TestApp:Settings:FontSize, Value= 24
* Key= TestApp:Settings:Message, Hello, World!

Read from the App Configuration store

Open `<app root>/Views/Home/Index.cshtml`, and replace its content with

````c#
@using Microsoft.Extensions.Configuration
@inject IConfiguration Configuration

<style>
    body {
        background-color: @Configuration["TestApp:Settings:BackgroundColor"]
    }
    h1 {
        color: @Configuration["TestApp:Settings:FontColor"];
        font-size: @Configuration["TestApp:Settings:FontSize"]px;
    }
</style>

<h1>@Configuration["TestApp:Settings:Message"]</h1>
````

Build and run the app locally

`dotnet build`

`dotnet run`

Open in browser `http://localhost:5000/`

### Manage Keys, Secrets and Certificates by using the Key Vault

Azure Key Vault helps solve the following problems:

* Secrets Management - Azure Key Vault can be used to Securely store and tightly control access to tokens, passwords, certificates, API keys, and other secrets
* Key Management - Azure Key Vault can also be used as a Key Management solution. Azure Key Vault makes it easy to create and control the encryption keys used to encrypt your data.
* Certificate Management - Azure Key Vault is also a service that lets you easily provision, manage, and deploy public and private Transport Layer Security/Secure Sockets Layer (TLS/SSL) certificates for use with Azure and your internal connected resources.

Stores

* HSM-protected keys (/keys)
* Software-protected keys (/keys)
* Secrets (/secrets)
* Certificates (/certificates)
* Storage account keys (/storageaccount)

#### Create a key vault using the Azure CLI

Create a resource group

`az group create --name "az204-app-configuration" -l "canadacentral"`

Create a key vault

`az keyvault create --name "az204learnkeyvault" --resource-group "az204-app-configuration" --location "canadacentral"`

Response includes the `Vault Name` and the `Vault URI` that applications using the vault access via the REST API.

````
Location       Name                ResourceGroup
-------------  ------------------  -----------------------
canadacentral  az204learnkeyvault  az204-app-configuration
````

Add a secret

Azure Key Vault > Settings > Secrets > Generate/Import

Show the secret

Select > Version Show Secret Value 




#### Securely save secret application settings for a web application

Traditionally all web application configuration settings are saved in configuration files such as Web.config. This practice leads to checking in secret settings such as Cloud credentials to public source control systems like GitHub.

Save secret settings in User Secret store that is outside of source control folder

If you are doing a quick prototype or you don't have internet access, start with moving your secret settings outside of source control folder to User Secret store. User Secret store is a file saved under user profiler folder, so secrets are not checked in to source control.

Save secret settings in Azure Key Vault

1. Create a Key Vault in your Azure subscription.
2. Grant you and your team members access to the Key Vault.
3. Add your secret to Key Vault on the Azure portal.
4. Add the NuGet packages `Azure.Identity` and `Azure.Extensions.AspNetCore.Configuration.Secrets` to your project
5. Update your program code:

````c#
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
    .ConfigureAppConfiguration((ctx, builder) =>
    {
        var keyVaultEndpoint = GetKeyVaultEndpoint();
        if (!string.IsNullOrEmpty(keyVaultEndpoint))
        {
            builder.AddAzureKeyVault(new Uri(keyVaultEndpoint), new DefaultAzureCredential(), new KeyVaultSecretManager());
        }
    })
    .ConfigureWebHostDefaults(webBuilder =>
    {
        webBuilder.UseStartup<Startup>();
    });

private static string GetKeyVaultEndpoint() => Environment.GetEnvironmentVariable("KEYVAULT_ENDPOINT");
````

6. Add your Key Vault URL to `launchsettings.json` file
7. Start debugging the project. It should run successfully.

ASP.NET and .NET applications

Save secret settings in a secret file that is outside of source control folder

Put secrets under the root element:

````xml
<?xml version="1.0" encoding="utf-8"?>
<root>
  <secrets ver="1.0">
    <secret name="secret" value="foo"/>
    <secret name="secret1" value="foo_one" />
    <secret name="secret2" value="foo_two" />
  </secrets>
</root>
````

Specify appSettings section is using the secret configuration builder. Make sure there is an entry for the secret setting with a dummy value.

````xml
<appSettings configBuilders="Secrets">
    <add key="webpages:Version" value="3.0.0.0" />
    <add key="webpages:Enabled" value="false" />
    <add key="ClientValidationEnabled" value="true" />
    <add key="UnobtrusiveJavaScriptEnabled" value="true" />
    <add key="secret" value="" />
</appSettings>
````

Save secret settings in an Azure Key Vault

Install the Nuget package `Microsoft.Configuration.ConfigurationBuilders.Azure`

Define Key Vault configuration builder in Web.config. Put this section before appSettings section. Replace vaultName to be the Key Vault name.

````xml
<configBuilders>
    <builders>
        <add name="Secrets" userSecretsId="695823c3-6921-4458-b60b-2b82bbd39b8d" type="Microsoft.Configuration.ConfigurationBuilders.UserSecretsConfigBuilder, Microsoft.Configuration.ConfigurationBuilders.UserSecrets, Version=2.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" />
        <add name="AzureKeyVault" vaultName="[VaultName]" type="Microsoft.Configuration.ConfigurationBuilders.AzureKeyVaultConfigBuilder, Microsoft.Configuration.ConfigurationBuilders.Azure, Version=2.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" />
    </builders>
</configBuilders>
````

Specify appSettings section is using the Key Vault configuration builder. Make sure there is any entry for the secret setting with a dummy value.

````xml
<appSettings configBuilders="AzureKeyVault">
    <add key="webpages:Version" value="3.0.0.0" />
    <add key="webpages:Enabled" value="false" />
    <add key="ClientValidationEnabled" value="true" />
    <add key="UnobtrusiveJavaScriptEnabled" value="true" />
    <add key="secret" value="" />
</appSettings>
````

### Implement Managed Identities for Azure resources

A common challenge for developers is the management of secrets and credentials to secure communication between different services.

* System-assigned Some Azure services allow you to enable a managed identity directly on a service instance. When you enable a system-assigned managed identity an identity is created in Azure AD that is tied to the lifecycle of that service instance.

* User-assigned You may also create a managed identity as a standalone Azure resource.

Build Applications on `Azure Resources` that access `Azure Services supporting AD Authentication` without needing to manage credentials.

#### How to use managed identities for App Service and Azure Functions

Add a system-assigned identity

Creating an app with a system-assigned identity requires an additional property to be set on the application.

Portal

1. Create an app in the portal as you normally would. Navigate to it in the portal.
2. If using a function app, navigate to Platform features. For other app types, scroll down to the Settings group in the left navigation.
3. Select Identity.
4. Within the System assigned tab, switch Status to On. Click Save.

CLI

Create a web application using the CLI.

````
az group create --name myResourceGroup --location westus
az appservice plan create --name myPlan --resource-group myResourceGroup --sku S1
az webapp create --name myApp --resource-group myResourceGroup --plan myPlan
````

Run the identity assign command to create the identity for this application

`az webapp identity assign --name myApp --resource-group myResourceGroup`

Azure Resource Manager template

Any resource of type `Microsoft.Web/sites` can be created with an identity by including in the resource definition

````json
"identity": {
    "type": "SystemAssigned"
}
````

Add a user-assigned identity

Similar to system, from Platform features > Settings > Identity

Within the User assigned tab, click Add.

Search for the identity you created earlier and select it. Click Add.

#### Obtain tokens for Azure resources

````c#
private readonly HttpClient _client;
// ...
public async Task<HttpResponseMessage> GetToken(string resource)  {
    var request = new HttpRequestMessage(HttpMethod.Get, 
        String.Format("{0}/?resource={1}&api-version=2019-08-01", Environment.GetEnvironmentVariable("IDENTITY_ENDPOINT"), resource));
    request.Headers.Add("X-IDENTITY-HEADER", Environment.GetEnvironmentVariable("IDENTITY_HEADER"));
    return await _client.SendAsync(request);
}
````

#### Secure Azure SQL Database connection from App Service using a managed identity

Grant database access to Azure AD user

`azureaduser=$(az ad user list --filter "userPrincipalName eq '<user-principal-name>'" --query [].objectId --output tsv)`

Add this Azure AD user as an Active Directory admin

`az sql server ad-admin create --resource-group myResourceGroup --server-name <server-name> --display-name ADMIN --object-id $azureaduser`

Modify ASP.NET Core

`Install-Package Microsoft.Azure.Services.AppAuthentication -Version 1.4.0`

With Active Directory authentication, you want both environments to use the same connection string. In `appsettings.json`, replace the value of the `MyDbConnection` connection string with:

`"Server=tcp:<server-name>.database.windows.net,1433;Database=<database-name>;"`

Supply the Entity Framework database context with the access token for the SQL Database.

````c#
var conn = (Microsoft.Data.SqlClient.SqlConnection)Database.GetDbConnection();
conn.AccessToken = (new Microsoft.Azure.Services.AppAuthentication.AzureServiceTokenProvider()).GetAccessTokenAsync("https://database.windows.net/").Result;
````
