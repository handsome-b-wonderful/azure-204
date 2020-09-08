# Develop for Azure Storage (10-15%)

## Develop Solutions That Use Cosmos DB Storage

### Select the appropriate API for your solution



### Implement partitioning schemes



### Interact with data using the appropriate SDK



### Set the appropriate consistency level for operations



### Create Cosmos DB containers



### Implement scaling (partitions, containers)



### Implement server-side programming including stored procedures, triggers, and change feed notifications


### Stored Procedures in Azure Cosmos DB


### Triggers in Azure Cosmos DB


### Change Feed Notifications





## Develop Solutions That Use Blob Storage

### Redundancy

#### __Redundancy in the primary region__

Data in an Azure Storage account is always replicated three times in the primary region

__Locally redundant storage (LRS)__

* copies data synchronously three times within a single physical location in the primary region
* least expensive replication option
* durability prone to location-wide failure
* not recommended for applications requiring high availability
* supported storage account types:
	* General-purpose v2
	* General-purpose v1
	* Block blob storage
	* Blob storage
	* File storage

__Zone-redundant storage (ZRS)__

* copies data synchronously across three Azure availability zones in the primary region
* availability zone is a separate physical location with independent power, cooling, and networking.
* applications requiring high availability: recommend ZRS in the primary region, and replicating to a * secondary region
* supported storage account types:
	* General-purpose v2
	* Block blob storage
	* File storage

#### __Redundancy in a secondary region__

copies the data in the storage account to a secondary region

__Geo-redundant storage (GRS)__

* copies data synchronously three times within a single physical location in the primary region using LRS then copies data asynchronously to a single physical location in the secondary region
* supported storage account types:
	* General-purpose v2
	* General-purpose v1
	* Blob storage

__Geo-zone-redundant storage (GZRS)__

* copies data synchronously across three Azure availability zones in the primary region using ZRS then copies data asynchronously to a single physical location in the secondary region
* supported storage account types:
	* General-purpose v2

For both data is always replicated synchronously three times using LRS in the secondary region
data in the secondary region isn't available for read or write access unless there is a failover to the secondary region
For read access to the secondary region, configure your storage account to use read-access geo-redundant storage (RA-GRS) or read-access geo-zone-redundant storage (RA-GZRS)

RA-GRS/RA-GZRS: Read access to the secondary region is available if the primary region becomes unavailable

### Move items in blob storage between storage accounts or containers

No explicit "moves"; copy & delete

__CLI using "az storage" commands__

* upload & download are synchronous operations
* no simple restart after a failure; repeat entire operation
* "az storage blob copy" is asynchronous, track progress, cancel and batch

__AzCopy utility for copying data into and out of Azure storage accounts__

* async and recoverable
* tune to match processing and bandwidth of local machine
* supports hierarchical containers and blob selection with pattern matching

__.NET Storage Client Library__

* programming interface to upload, download and migrate blobs between storage accounts
* existing applications or various deployment environments (ex: Azure Functions)
* blob selection based on metadata

__Using the Command Line Interface (CLI)__

* create storage account

	`az storage account create --location uswest2 --name myAccount --resource-group myGroup --sku Standard_RAGRS --kind BlobStorage --access-tier hot`

* get keys

	`az storage account keys list --account-name myAccount --resource-group myGroup --output table`

* create storage container

	`az storage container create --name myContainer --account-name myAccount --account-key [storage account key]`

* upload blob

	`az storage blob upload --container-name myContainer --name myBlob --file blobdata.dat --account-name myAccount --account-key [storage account key]`

	OR

	```
	// store account info in environment variables that will be referenced by default
	AZURE_STORAGE_ACCOUNT = myAccount
	AZURE_STORAGE_KEY  = [storage account key]
	az storage blob upload --container-name myContainer --name myBlob --file blobdata.dat
	```

* overwrites existing blobs named "myBlob"
* ETag filtering

	* --if-none-match overwrites the blob if none of the ETags supplied in the command match that of the blob
	* --if-modified-since overwrites only if it has been modified since a specified date
	* --if-unmodified-since ensures the blob hasn't changed since the date given

* Uploads if blob with same name has not changed since specified date:

	`az storage blob upload --container-name myContainer --name myBlob --file blobdata.dat --if-unmodified-since 2019-05-26T10:30Z`

* Batch Upload

	`az storage blob upload-batch --destination myCOntainer --source myFolder --pattern *.png`

* List blobs in a container

	`az storage blob list --container-name myContainer --output table`

* Copy Blobs between Accounts

	`az storage blob copy start --destination-container destContainer --destination-blob myBlob --source-account-name mySourceAccount --source-account-key mySourceAccountKey --source-container myContainer --source-blob myBlob`

	OR

	`az storage blob copy start --destination-container destContainer --destination-blob myBlob --source-uri myBlobUrl`

* runs async - check status

	`az storage blob show --container-anme DestContainer --name myBlob`

* ETags for destination and source

	* --destination-if-match
	* --destination-if-none-match
	* --destination-if-modified-since
	* --destination-if-unmodified-since
	* --source-if-match
	* --source-if-none-match
	* --source-if-modified-since
	* --source-if-unmodified-since

* Batch Copy

	`az storage blob copy start-batch --destination-container destContainer --source-account-name mySourceAccount --source-account-key mySourceAccountKey --source-container myContainer --pattern *.dat`

* Check on Status

	`az storage blob list`

* Delete a Blob

	`az storage blob delete --container sourceContainer --name sourceBlob`

* Batch Delete

	`az storage blob delete-batch --source sourceContainer`

* ETag filtering available
* Delete blobs that have not been modified in the last six months

	```
	date=`date -d "6 months ago" '+%Y-%m-%dT%H:%MZ'`
	az storage blob delete-batch --source sourceContainer --if-unmodified-since $date
	```

__Using AzCopy__

* command line utility optimized for moving data into and out of Azure storage
* bulk transfer operations

	`azcopy login` OR use Shared Access Signature (SAS) configured to desired target(s), lifetime and operations

	`azcopy copy "myfile.txt" "https://myacount.blobb.core.windows.net/mycontainer/"`

* Multiple files and folders

	`azcopy copy "myfolder" "target-container-url" --recursive=true`

* Review current jobs

	```
	azcopy jobs list
	azcopy jobs show <id>
	```

* Restart

	`azcopy jobs resume <id>`

* Download

	`azcopy copy "sourceBlobUri" "myblobdata"`

* Transfer between storage accounts

	`azcopy copy "https://sourceaccount.blob.core.windows.net/sourcecontainer/*?<source sas token>" "https://destaccount.blob.core.windows.net/destcontainer/*?<dest sas token>"`

	* can add --recursive=true for hierarchical set of blobs

* Sync files

	`azcopy sync "source" destination"`

	* if not found in destination or if older than the source
	* use `--delete-destination` to delete blobs in the destination that do not exist in source

* Manage Blobs

	`azcopy list "https://sourceaccount.blob.core.windows.net/sourcecontainer?<sas token>"`

* Create a container

	`azcopy make "https://myaccount.blob.core.windows.net/newcontainer?<sas token>"`

* Remove Blobs

	`azcopy remove "uri"
	--include <pattern>
	--recursive=true`

		* AZCOPY_CONCURRENCY_VALUE  (default 300) sets # of concurrent threads for transfers

__Blob .NET Client__

* Connect to storage

```C#
	using Microsoft.WindowsAzure.Storage;
	using Microsoft.WindowsAzure.Storage.Blob;

	// The variable sourceConnection is a string holding the connection string for the storage account
	CloudStorageAccount sourceAccount = CloudStorageAccount.Parse(sourceConnection);
	CloudBlobClient sourceClient = sourceAccount.CreateCloudBlobClient();
```
	
* Download

```C#
	CloudBlobContainer sourceBlobContainer = sourceClient.GetContainerReference(sourceContainer);
	ICloudBlob sourceBlob = await sourceBlobContainer.GetBlobReferenceFromServerAsync("MyBlob.doc");
	Console.WriteLine($"Last modified: {sourceBlob.Properties.LastModified}");
	await sourceBlob.DownloadToFileAsync("MyFile.doc", System.IO.FileMode.Create);
```
	
* Upload

```C#
	CloudBlobContainer destBlobContainer = destClient.GetContainerReference(destContainer);
	CloudBlockBlob destBlob = destBlobContainer.GetBlockBlobReference("NewBlob.doc");
	await destBlob.UploadFromFileAsync("MyFile.doc");
```

* Copy between accounts

```C#
	CloudBlockBlob destBlob = destContainer.GetBlockBlobReference(sourceBlob.Name);
	await destBlob.StartCopyAsync(new Uri(GetSharedAccessUri(sourceBlob.Name, sourceContainer)));

	// get a SAS for read permissions on a specific blob for a limited period of time
	// create policy
	SharedAccessBlobPolicy policy = new SharedAccessBlobPolicy{Permissions = SharedAccessBlobPermissions.Read, SharedAccessStartTime = null,SharedAccessExpiryTime = new DateTimeOffset(toDateTime)};
	// get blob reference
	CloudBlockBlob blob = container.GetBlockBlobReference(blobName);
	// get SAS
	string sas = blob.GetSharedAccessSignature(policy);
	// append SAS to URI
	return blob.Uri.AbsoluteUri + sas;
```

* Delete Blob

```C#
	bool blobExisted = await sourceBlob.DeleteIfExistsAsync();

	Iterate Blobs in container
	// in loop repeat this:
	ICloudBlob blob = await blobContainer.GetBlobReferenceFromServerAsync(blobItem.Name)
```

### Set and retrieve properties and metadata

* Blob containers support
	* system properties (read/write or read-only)
		* some correspond to specific HTTP headers
	* user-defined metadata (name-value pairs)
		* no impact on resource behaviour

	* GetProperties / GetPropertiesAsync
		
		```C#
		 var properties = await container.GetPropertiesAsync();
        Console.WriteLine($"Properties for container {container.Uri}");
        Console.WriteLine($"Public access level: {properties.Value.PublicAccess}");
        Console.WriteLine($"Last modified time in UTC: {properties.Value.LastModified}");
		```
	
    * SetMetadata / SetMetadataAsync
		* duplicate metadata is comma-separated and concatenated (returns 200 - OK)

		```C#
		IDictionary<string, string> metadata = new Dictionary<string, string>();
        // Add some metadata to the container.
        metadata.Add("docType", "textDocuments");
        metadata.Add("category", "guidance");
		```

    * GetProperties / GetPropertiesAsync

	```C#
		var properties = await container.GetPropertiesAsync();
		foreach (var metadataItem in properties.Value.Metadata)
		{
			Console.WriteLine($"\tKey: {metadataItem.Key}");
            Console.WriteLine($"\tValue: {metadataItem.Value}");
        }
	```

### Interact with data using the appropriate SDK

* __Complete the sample projects using the C# SDK___

### Implement data archiving and retention

#### Store business-critical blob data with immutable storage

* WORM: Write Once Read Many state
* non-erasable and non-modifiable for a user-specified interval
* Blobs can be created and read but not modified or deleted for the duration of the retention interval
* Available in all regions for general-purpose v1, general-purpose v2, BlobStorage, and BlockBlobStorage
* Regulatory compliance, secure document retention, legal hold
* time-based retention policy: locked for SEC and other compliance; unlocked for testing
* legal hold policy
* independent on all blob tiers (hot, cool, archive)
* container-level configuration
* container-level policy audit log
* allowProtectedAppendWrites setting is available for time-based policies applied to append blobs
* the retention period for the blob is the defined life from the latest block appended
* Container can have both legal and time retention policies
* legal holds:
	* max of 10,000 containers with a legal hold setting per storage account
	* max 10 legal hold tags per container
	* length of a legal hold tag 3-23 alphanumeric chars
	* max 10 legal hold policy audit logs retained for the duration of the policy, per container

#### Rehydrate blob data from the archive tier

* archive access blobs are offline and cannot be read or modified
* blob metadata is online and available
* Accessing Archived blobs:
	1. Rehydrate an archived blob to an online tier (Set Blob Tier) operation
	2. Copy an archived blob (Copy Blob) to a new blob name and a destination tier of hot or cool
* Rehydrate priority:
* Standard priority - order received, up to 15 hours
* High priority - before standard, maybe under an hour (depending on blob size and current demand)

### Implement hot, cool, and archive storage

* Azure Blob storage: hot, cool, and archive access tiers
	* Hot tier: frequent access
		* in active use, expected frequent read/writes
		* staged for processing or migration to cool tier
	* Cool tier: infrequent access & stored for at least 30 days
		* short-term back-up and recovery data sets
		* older media content not viewed frequently but requiring immediate availability
		* large data sets that need to be cost effective storage while more data is being gathered (scientific data, raw telemetry)
	* Archive tier: rare access, stored at least 180 days and flexible latency (hours)
		* < 180 days == early deletion charge
		* offline; must first be rehydrated
		* metadata remains online and availably: GetBlobProperties, GetBlobMetadata, SetBlobTags, GetBlobTags, * FindBlobsByTags, ListBlobs, SetBlobTier, CopyBlob, DeleteBlob
		* long-term backup, secondary backup or archival
		* original data for preservation after processing
		* Compliance and archival data
			* not supported in ZRS, GZRS or RA-GZRS accounts

	* Hot/Cool set at account level
	* All set at blob level during or after upload
	* Cool tier - slightly lower availability but high durability, retrieval latency and throughput similar to hot
		* lower availability SLA and higher access costs vs. lower storage costs
	* Archive tier - offline with lowest storage costs and highest rehydrate & access costs
	
	* tiering only supported in Blob storage and General Purpose v2 (GPv2) accounts
	* Data stored in a block blob storage account (Premium performance) cannot be tiered - move first to the hot tier of a different account
	* General Purpose v1 (GPv1) accounts do not support tiering and need to be converted
	* Blob storage and GPv2 accounts expose the Access Tier attribute at the account level.
		* default access tier for any blob not explicitly set at the object level
		* archive only applies at object level
		* access tier can be changed at any time
		* account-level tiering populates the "Access Tier Inferred" blob property (hot or cool)
		* Blob-level set with PutBlob, PutBlockList or SetBlobTier operations
	* Blob lifecycle management: rule-based policy to transition data to the best tier and expire data
	* move to cooler tier: billed as write operation (per 10,000) and data write (per GB)
	* move to warmer tier: read operation (per 10,000) and data retrieval (per GB) + early deletion charges if applicable (prorated)
* 	Objects in the cool tier on GPv2 accounts have a minimum retention duration of 30 days. Blob storage accounts don't have a minimum retention duration for the cool tier.
	* Archive Storage currently supports 2 rehydrate priorities (High and Standard) that offers different retrieval latencies. For more information
	* Recommend you use GPv2 instead of Blob storage accounts for tiering. GPv2 support all the features that * Blob storage accounts support plus a lot more. Pricing between Blob storage and GPv2 is almost identical, but some new features and price cuts will only be available on GPv2 accounts. GPv1 accounts don't support tiering.
	
