# Monitor, Troubleshoot, and Optimize Azure Solutions (10-15%)

## Integrate Caching and Content Delivery within Solutions

Content delivery network on Azure

A content delivery network (CDN) is a distributed network of servers that can efficiently deliver web content to users. CDNs' store cached content on edge servers in point-of-presence (POP) locations that are close to end users, to minimize latency.

Azure CDN features

* Dynamic site acceleration (vs traditional static caching)
* CDN caching rules
* HTTPS custom domain support
* Azure diagnostics logs: view core analytics and save them to Azure  storage account, log analytics workspace or Azure event hub
* File compression (origin server or @ edge "on the fly")
* Geo-filtering: Restrict Azure CDN content by country/region

### Develop code to implement CDN’s in solutions

#### Storage Account CDN

Create resource group

`az group create --name az204-learn-cdn --location canadacentral`

Create a storage account

`az storage account create --name az204cachestorageacct --location canadacentral --resource-group az204-learn-cdn`

Create a profile

`az cdn profile create --location canadacentral --name az204cdn --resource-group az204-learn-cdn --sku Standard_Microsoft`

Create a CDN endpoint

`az cdn endpoint create --resource-group az204-learn-cdn --profile-name az204cdn --name az204learncdn.azureedge.net --origin az204cachestorageacct.blob.core.windows.net`

Access Content

`http://az204cachestorageacct.blob.core.azureedge.net/<myPublicContainer>/<BlobName>`

example:

`https://az204cachestorageacct.blob.core.windows.net/cache/hang.jpg` is available via CDN at `http://az204cachestorageacct.blob.core.azureedge.net/cache/hang.jpg`

Remove content from Azure CDN

* Make the container private instead of public.
* Disable or delete the CDN endpoint by using the Azure portal.
* Modify your hosted service to no longer respond to requests for the object.

#### Get started with the Azure CDN Library for .NET

Create a resource group

Create an Azure AD application and apply permissions

Use Interactive user authentication instead of a service principal

Create a project and add Nuget packages

`Microsoft.IdentityModel.Clients.ActiveDirectory`

`Microsoft.Azure.Management.Cdn`

List CDN profiles and endpoints

````c#
private static void ListProfilesAndEndpoints(CdnManagementClient cdn)
{
    // List all the CDN profiles in this resource group
    var profileList = cdn.Profiles.ListByResourceGroup(resourceGroupName);
    foreach (Profile p in profileList)
    {
        Console.WriteLine("CDN profile {0}", p.Name);
        if (p.Name.Equals(profileName, StringComparison.OrdinalIgnoreCase))
        {
            // Hey, that's the name of the CDN profile we want to create!
            profileAlreadyExists = true;
        }

        //List all the CDN endpoints on this CDN profile
        Console.WriteLine("Endpoints:");
        var endpointList = cdn.Endpoints.ListByProfile(p.Name, resourceGroupName);
        foreach (Endpoint e in endpointList)
        {
            Console.WriteLine("-{0} ({1})", e.Name, e.HostName);
            if (e.Name.Equals(endpointName, StringComparison.OrdinalIgnoreCase))
            {
                // The unique endpoint name already exists.
                endpointAlreadyExists = true;
            }
        }
        Console.WriteLine();
    }
}
````

#### Create a new CDN endpoint

CDN profiles > profile > endpoints

* Name: unique name
* Origin type: Storage, Cloud  service, Web App, Custom origin (other)
* Origin hostname: drop-down lists all available origin servers of the type specified
* Origin path: path to the resources to be cached
* Origin host header: host header to be sent with each request
* Protocol and Origin port: protocols and ports to use to access resources at the origin server
* Optimized for: targeted scenario / content type

Scenarios:

* Azure CDN Standard from Microsoft
    * General web delivery (including media streaming and large file download)
    * Dynamic site acceleration from Microsoft is offered via Azure Front Door Service
* Azure CDN Standard from Verizon and Azure CDN Premium from Verizon
    * General web delivery (including media streaming and large file download)
    * Dynamic site acceleration
* Azure CDN Standard from Akamai profiles
    * General web delivery
    * General media streaming
    * Video-on-demand media streaming
    * Large file download
    * Dynamic site acceleration

#### Azure Front Door

Azure Front Door is a global, scalable entry-point that uses the Microsoft global edge network to create fast, secure, and widely scalable web applications.

Create a Front Door with Azure CLI

Add the Front Door extension to the Azure CLI

`az extension add --name front-door`

Create two resource groups

`az group create --name az204-learn-frontdoor --location canadacentral`

`az group create --name az204-learn-backdoor --location westus`

Create two app service plans

`az appservice plan create --name myAppServiceCanadaCentral --resource-group az204-learn-frontdoor`

`az appservice plan create --name myAppServicePlanWestUS --resource-group az204-learn-backdoor`

Create two web apps

`az webapp create --name WebAppFrontside --resource-group az204-learn-frontdoor --plan myAppServiceCanadaCentral`

`az webapp create --name WebAppBackside --resource-group az204-learn-backdoor --plan myAppServicePlanWestUS`

Create the Front Door

`az network front-door create --resource-group az204-learn-frontdoor --name webapp-frontend --accepted-protocols http https --backend-address webappfrontside.azurewebsites.net webappbackside.azurewebsites.net`

Test the Front Door

Open a web browser and enter the hostname obtain from the commands. The Front Door will direct your request to one of the backend resources.

Clean up resources

`az group delete --name az204-learn-frontdoor`

`az group delete --name az204-learn-backdoor`

#### Azure Cache for Redis

Azure Cache for Redis provides an in-memory data store based on the open-source software Redis. Redis improves the performance and scalability of an application that uses on backend data stores heavily.

Key scenarios

* Data cache
* Content cache
* Session store
* Job and message queuing
* Distributed transactions

#### Considerations of caching

* CDN fallback if not available (local cache, other sources)
* Delivery over HTTPS to avoid mixed-content warnings from the browser
* Cache expiry and versioning
* Cache control - rules on endpoints

### Configure cache and expiration policies for FrontDoor, CDNs, or Redis caches Store and retrieve data in Azure Redis cache






#### Redis

Redis supports both read and write operations.

Writes can be protected from system failure either by being stored periodically in a local snapshot file or in an append-only log file.

Redis is a key-value store, where values can contain simple types or complex data structures such as hashes, lists, and sets.

Redis enables a client application to submit a series of operations that read and write data in the cache as an atomic transaction.

* guaranteed to run sequentially
* no commands issued by other concurrent clients will be interwoven between them

Runs inside a trusted environment that can be accessed only by trusted clients.
 
Redis supports a limited security model based on password authentication.

### Azure Redis cache

Redis servers that are hosted at an Azure datacenter.

Façade that provides access control and security.

The simplest use of Redis for caching concerns is key-value pairs where the value is an uninterpreted string of arbitrary length that can contain any binary data.

Redis supports a series of atomic get-and-set operations on string values.

Fire and forget cache operations

````c#
ConnectionMultiplexer redisHostConnection = ...;
IDatabase cache = redisHostConnection.GetDatabase();
...
await cache.StringSetAsync("data:key1", 99);
...
cache.StringIncrement("data:key1", flags: CommandFlags.FireAndForget);
````

#### Control Azure CDN caching behavior with caching rules

Control file caching:

* Caching rules: set or modify default cache expiration behavior both globally and with custom conditions, such as a URL path and file extension.
    * Global caching rules: You can set one global caching rule for each endpoint in your profile.
    * Custom caching rules: You can set one or more custom caching rules for each endpoint in your profile.
        * match specific paths and file extensions
        * take precedence over global rules
* Query string caching: You can adjust how the Azure CDN treats caching for requests with query strings.

CDN Profiles > profile > endpoint > Caching rules

* Bypass cache: Do not cache and ignore origin-provided cache-directive headers.
* Override: Ignore origin-provided cache duration; use the provided cache duration instead. This will not override cache-control: no-cache.
* Set if missing: Honor origin-provided cache-directive headers, if they exist; otherwise, use the provided cache duration.

#### Caching with Azure Front Door

Azure Front Door delivers large files without a cap on file size using "object chunking", requesting the file from the backend in chunks of 8 MB.

After the chunk arrives at the Front Door environment, it's cached and immediately served to the user. Front Door then pre-fetches the next chunk in parallel. This pre-fetch ensures that the content stays one chunk ahead of the user, which reduces latency.

Query string behavior

* Ignore query strings: In this mode, Front Door passes the query strings from the requestor to the backend on the first request and caches the asset. All ensuing requests for the asset that are served from the Front Door environment ignore the query strings until the cached asset expires.
* Cache every unique URL: In this mode, each request with a unique URL, including the query string, is treated as a unique asset with its own cache.

Front Door caches assets until the asset's time-to-live (TTL) expires

* Single path purge: Purge individual assets by specifying the full path of the asset (without the protocol and domain), with the file extension, for example, /pictures/strasbourg.png;
* Wildcard purge: Asterisk * may be used as a wildcard. Purge all folders, subfolders, and files under an endpoint with /* in the path or purge all subfolders and files under a specific folder by specifying the folder followed by `/*`, for example, `/pictures/*`.
* Root domain purge: Purge the root of the endpoint with "/" in the path.

Cache expiration - order of headers

1. `Cache-Control: s-maxage=<seconds>`
2. `Cache-Control: max-age=<seconds>`
3. `Expires: <http-date>`

Request headers not forwarded:

* `Content-Length`
* `Transfer-Encoding`

#### Azure Cache for Redis

search "Azure Cache for Redis" > Create Redis cache

Key values are DNS name (global unique), pricing, connectivity method and "enable a non-TLS port"

Best practices for Azure Cache for Redis

* Use Standard or Premium tier for production systems. The Basic tier is a single node system with no data replication and no SLA.
* Remember that Redis is an in-memory data store
* Develop your system such that it can handle connection blips
* Configure your maxmemory-reserved setting to improve system responsiveness under memory pressure conditions
* Redis works best with smaller values, so consider chopping up bigger data into multiple keys
* Locate your cache instance and your application in the same region.
* Reuse connections. Creating new connections is expensive and increases latency.
* Configure your client library to use a connect timeout of at least 15 seconds, giving the system time to connect even under higher CPU conditions.

Retrieve host name, ports, and access keys from the Azure portal

Azure Cache for Redis > Access Keys > primary

Azure Cache for Redis > Properties > host name

#### Azure Cache for Redis with a .NET Core app

Create a console app

`dotnet new console -o Redistest`

Add Secret Manager to the project

```xml
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <OutputType>Exe</OutputType>
        <TargetFramework>netcoreapp2.0</TargetFramework>
        <UserSecretsId>ANY_GUID</UserSecretsId>
    </PropertyGroup>
    <ItemGroup>
        <DotNetCliToolReference Include="Microsoft.Extensions.SecretManager.Tools" Version="2.0.0" />
    </ItemGroup>
</Project>
```

Add extension to project

`dotnet add package Microsoft.Extensions.Configuration.UserSecrets`

Update `Program.cs`

`using Microsoft.Extensions.Configuration;`

 Initialize a configuration to access the user secret for the Azure Cache for Redis connection string in the Program class in `Program.cs`

 ````c#
 private static IConfigurationRoot Configuration { get; set; }
const string SecretName = "CacheConnection";

private static void InitializeConfiguration()
{
    var builder = new ConfigurationBuilder()
        .AddUserSecrets<Program>();

    Configuration = builder.Build();
}
 ````

Configure the cache client

`dotnet add package StackExchange.Redis`

Connect to the cache

````c#
using StackExchange.Redis;

// in the Program class

private static Lazy<ConnectionMultiplexer> lazyConnection = new Lazy<ConnectionMultiplexer>(() =>
{
    string cacheConnection = Configuration[SecretName];
    return ConnectionMultiplexer.Connect(cacheConnection);
});

public static ConnectionMultiplexer Connection
{
    get
    {
        return lazyConnection.Value;
    }
}
````

Executing cache commands

In `Program.cs`, add the following code for the `Main` procedure of the `Program` class

````c#
static void Main(string[] args)
{
    InitializeConfiguration();

    // Connection refers to a property that returns a ConnectionMultiplexer
    // as shown in the previous example.
    IDatabase cache = lazyConnection.Value.GetDatabase();

    // Perform cache operations using the cache object...

    // Simple PING command
    string cacheCommand = "PING";
    Console.WriteLine("\nCache command  : " + cacheCommand);
    Console.WriteLine("Cache response : " + cache.Execute(cacheCommand).ToString());

    ...

    lazyConnection.Value.Dispose();
}
````

Build & Run

````
> dotnet build

> dotnet run
````

Azure Cache for Redis can cache both .NET objects and primitive data types, but before a .NET object can be cached it must be serialized. This .NET object serialization is the responsibility of the application developer, and gives the developer flexibility in the choice of the serializer.

`dotnet add package Newtonsoft.json`

````c#
using Newtonsoft.Json;

...

class Employee
{
    public string Id { get; set; }
    public string Name { get; set; }
    public int Age { get; set; }

    public Employee(string EmployeeId, string Name, int Age)
    {
        this.Id = EmployeeId;
        this.Name = Name;
        this.Age = Age;
    }
}

...

// At the bottom of Main() procedure in Program.cs, and before the call to Dispose()

// Store .NET object to cache
Employee e007 = new Employee("007", "Davide Columbo", 100);
Console.WriteLine("Cache response from storing Employee .NET object : " + 
cache.StringSet("e007", JsonConvert.SerializeObject(e007)));

// Retrieve .NET object from cache
Employee e007FromCache = JsonConvert.DeserializeObject<Employee>(cache.StringGet("e007"));
Console.WriteLine("Deserialized Employee .NET object :\n");
Console.WriteLine("\tEmployee.Name : " + e007FromCache.Name);
Console.WriteLine("\tEmployee.Id   : " + e007FromCache.Id);
Console.WriteLine("\tEmployee.Age  : " + e007FromCache.Age + "\n");
````

Build & Run

````
> dotnet build

> dotnet run
````

## Instrument Solutions to Support Monitoring and Logging

### Azure Monitor Overview

Azure Monitor increases availability and performance of applications and services with collecting, analyzing, and acting on telemetry data.

* Detect and diagnose issues across applications and dependencies with Application Insights.
* Correlate infrastructure issues with Azure Monitor for VMs and Azure Monitor for Containers.
* Drill into your monitoring data with Log Analytics for troubleshooting and deep diagnostics.
* Support operations at scale with smart alerts and automated actions.
* Create visualizations with Azure dashboards and workbooks.

![Azure Monitor](img/azure-monitor-overview.png)

All data is:

* Metrics: numerical values that describe some aspect of a system at a particular point in time.
* Logs: different kinds of data organized into records with different sets of properties for each type. Telemetry such as events and traces are stored as logs in addition to performance data so that it can all be combined for analysis.

Azure Monitor collects

* Application monitoring data: Data about the performance and functionality of the code you have written, regardless of its platform.
* Guest OS monitoring data: Data about the operating system on which your application is running. This could be running in Azure, another cloud, or on-premises.
* Azure resource monitoring data: Data about the operation of an Azure resource.
* Azure subscription monitoring data: Data about the operation and management of an Azure subscription, as well as data about the health and operation of Azure itself.
* Azure tenant monitoring data: Data about the operation of tenant-level Azure services, such as Azure Active Directory.

Monitor provides analysis of the operation of a particular Azure application or service.

* Activity Log
* Alerts
* Metrics
* Logs
* Service Health
* Workbooks (builsing visualizations)

Insights:

* Applications
* VMs
* Storage Accounts
* Cosmos DB
* Key Vaults
* Azure Cache for Redis

### Configure instrumentation in an app or service by using Application Insights

Application Insights monitors the availability, performance, and usage of your web applications whether they're hosted in the cloud or on-premises.

#### Start monitoring in the Azure portal

Application Insights > Create

Add name, resouce group, location and "classic" type

Copy the instrumentation key

Overview > Instrumentation Key

Create a simple web page and add a reference to the JS SDK for App Insights

````js
<script type="text/javascript">
!function(T,l,y){var S=T.location,u="script",k="instrumentationKey",D="ingestionendpoint",

...

trackPageView({})}(window,document,{
src: "https://az416426.vo.msecnd.net/scripts/b/ai.2.min.js", // The SDK URL Source
//name: "appInsights", // Global SDK Instance name defaults to "appInsights" when not supplied
//ld: 0, // Defines the load delay (in ms) before attempting to load the sdk. -1 = block page load and add to head. (default) = 0ms load after timeout,
//useXhr: 1, // Use XHR instead of fetch to report failures (if available),
//crossOrigin: "anonymous", // When supplied this will add the provided value as the cross origin attribute on the script tag 
cfg: { // Application Insights Configuration
    instrumentationKey: "YOUR_INSTRUMENTATION_KEY_GOES_HERE"
    /* ...Other Configuration Options... */
}});
</script>
````

Visit the page a few times in a local browser session.

Monitor your website in the Azure portal

Monitoring > Logs > Start Query

````kusto
// average pageView duration by name
let timeGrain=1s;
let dataset=pageViews
// additional filters can be applied here
| where timestamp > ago(15m)
| where client_Type == "Browser" ;
// calculate average pageView duration for all pageViews
dataset
| summarize avg(duration) by bin(timestamp, timeGrain)
| extend pageView='Overall'
// render result in a chart
| render timechart
````

Overview > Investigate > Performance

Usage > Users > user behavior analytics tools

User Flows: track the pathway that visitors take through the various parts of your website.

#### Application Insights for ASP.NET Core applications

Start with a .NET core application

Install the Nuget package `Microsoft.ApplicationInsights.AspNetCore` and update your `.csproj` file

````xml
<ItemGroup>
    <PackageReference Include="Microsoft.ApplicationInsights.AspNetCore" Version="2.13.1" />
</ItemGroup>
````

Add `services.AddApplicationInsightsTelemetry();` to the `ConfigureServices()` method in `Startup`

````c#
// This method gets called by the runtime. Use this method to add services to the container.
public void ConfigureServices(IServiceCollection services)
{
    // The following line enables Application Insights telemetry collection.
    services.AddApplicationInsightsTelemetry();

    // This code adds other services for your application.
    services.AddMvc();
}
````

Update `appsettings.json` to add the instrumentation key and any custom logging level

````json
{
"ApplicationInsights": {
        "InstrumentationKey": "YOUR_INSTRUMENTATION_KEY",
		  "LogLevel": {
        "Default": "Trace"
      }
      },
  "Logging": {

...
````

Run the application and visit in the browser.

Refresh the Application Insights overview page

Overview > Server response time -or- Server requests

drill-down

You can modify a few common settings by passing `ApplicationInsightsServiceOptions` to `AddApplicationInsightsTelemetry` including:

* EnablePerformanceCounterCollectionModule
* EnableRequestTrackingTelemetryModule
* EnableEventCounterCollectionModule

and more. They all default to `true`

Add telemetry initializers when you want to enrich telemetry with additional information.

### Analyze log data and troubleshoot solutions by using Azure Monitor

#### Log Queries

* Adhoc queries
* Alert rules. Alert rules proactively identify issues from data in your workspace. Each alert rule is based on a log search that is automatically run at regular intervals. The results are inspected to determine if an alert should be created.
* Dashboards. You can pin the results of any query into an Azure dashboard which allow you to visualize log and metric data together and optionally share with other Azure users.
* Views. You can create visualizations of data to be included in user dashboards with View Designer. Log queries provide the data used by tiles and visualization parts in each view.
* Export. When you import log data from Azure Monitor into Excel or Power BI, you create a log query to define the data to export.
* PowerShell. You can run a PowerShell script from a command line or an Azure Automation runbook that uses Get-AzOperationalInsightsSearchResults to retrieve log data from Azure Monitor. This cmdlet requires a query to determine the data to retrieve.
* Azure Monitor Logs API. The Azure Monitor Logs API allows any REST API client to retrieve log data from the workspace. The API request includes a query that is run against Azure Monitor to determine the data to retrieve.

The Activity log is a platform log in Azure that provides insight into subscription-level events. This includes such information as when a resource is modified or when a virtual machine is started.

#### Find and diagnose performance issues with Azure Application Insights

Identify slow server operations

Application Insights > Performance > Server Response Time

Identify slow client operation

Browser > Investigate > Browser Performance > Page View Properties

Use logs data for client

View in Logs (Analytics) - write custom queries against the log data

#### Find and diagnose run-time exceptions with Azure Application Insights

Application Insights > Investigate > Failures > Failed request

The Failed requests panel shows the count of failed requests and the number of users affected for each operation for the application.

The Gantt chart will show any dependency failures in the transaction.

Identify failing code

The Snapshot Debugger collects snapshots of the most frequent exceptions in the application.

Exception Properties > Open debug snapshot

 View the call stack for the request. Click any method to view the values of all local variables at the time of the request.

Download Snapshot > Open in Visual Studio > Identify code causing the exception.

Use Analytics Data

All data collected by Application Insights is stored in Azure Log Analytics and can be viewed with `Analyze impact`

Add work item

Exception Properties > New Work Item

Save to DevOps, GitHub, etc.

### Implement Application Insights Web Test and Alerts

#### Web Tests

Create a URL ping test

Availability > Create Test

Set URL, HTTP response code indicating success, etc.

Will follow up to 10 redirects.

#### Alerts

Alerts proactively notify you when issues are found with your infrastructure or application using your monitoring data in Azure Monitor.

* Target Resource: Defines the scope and signals available for alerting.
* Signal: Emitted by the target resource. Signals can be of the following types: metric, activity log, Application Insights, and log.
* Criteria: A combination of signal and logic applied on a target resource
* Action: A specific action taken when the alert is fired

Types of Alerts

* Metric alerts: work on top of multi-dimensional metrics including platform metrics, custom metrics, popular logs from Azure Monitor converted to metrics and Application Insights metrics. Metric alerts evaluate at regular intervals to check if conditions on one or more metric time-series are true and notify you when the evaluations are met. Metric alerts are stateful, that is, they only send out notifications when the state changes.
* Log alerts: use a Log Analytics query to evaluate resources logs every set frequency, and fire an alert based on the results.
* Activity log alerts: activate when a new activity log event occurs that matches the conditions specified in the alert

Availability alerts

Sends web requests to your application at regular intervals from points around the world. It can alert you if your application isn't responding, or if it responds too slowly.

### Implement code that handles transient faults

How to handle temporary service interruptions?

#### Use smart retry/back-off logic to mitigate the effect of transient failures

Instead of throwing an exception and displaying a not available or error page to your customer, you can recognize errors that are typically transient, and automatically retry the operation that resulted in the error,

#### Circuit breakers

Prevent too many retries because:

* Too many users persistently retrying failed requests might degrade other users' experience. If millions of people are all making repeated retry requests you could be tying up IIS dispatch queues and preventing your app from servicing requests that it otherwise could handle successfully.
* If everyone is retrying due to a service failure, there could be so many requests queued up that the service gets flooded when it starts to recover.
* If the error is due to throttling and there's a window of time the service uses for throttling, continued retries could move that window out and cause the throttling to continue.
* You might have a user waiting for a web page to render. Making people wait too long might be more annoying that relatively quickly advising them to try again later.

Exponential back-off addresses some of these issues, but a circuit breaker means you stop retrying, period.

* Custom fallback to backup source or cache
* Fail silent if the operation is non-transactional
* Fail fast to error out the user to avoid flooding the service with retry requests
