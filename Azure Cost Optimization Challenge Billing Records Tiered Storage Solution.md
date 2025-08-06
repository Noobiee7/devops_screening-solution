# Azure Cost Optimization Challenge: Billing Records Tiered Storage Solution

**Date:** Wednesday, August 06, 2025
**By:** Rohit Lakkabathini

## 1. Problem Statement \& Context

- The system stores over 2 million billing records in Azure Cosmos DB.
- Each record can be up to 300 KB in size.
- The system is read-heavy, but records older than 3 months are rarely accessed.
- The database size and associated cost have grown significantly over time.
- Goal: Optimize costs while ensuring availability and minimal latency (seconds) for old records.
- Constraints:
    - No downtime or data loss during migration.
    - No changes to existing API contracts.
    - Solution must be simple and maintainable.


## 2. Solution for Managing Billing Records in Azure Serverless Architecture.

### 1. Simplicity & ease of implementation

Azure Function (Timer Triggered): Archives old data (older than 3 months) to Azure Blob Storage (Cool Tier) stores old records as compressed JSON. Schedule a daily/weekly job.
This approach requires minimal changes and is easy to deploy and maintain.

### 2. No Data loss and No Downtime 

Implement a two-phase archival process:
Azure Function reads and verifies records before deleting from Cosmos DB. Each record is saved in Blob Storage and validated (e.g., using checksum). No hard deletes until successful archival confirmation. Cosmos DB remains active and migration is asynchronous with no service interruption.

### 3. No change in API

On read: First check Cosmos DB, if not found, fetch from Blob Storage.
On write: Continue writing to Cosmos DB as before. 
Add 'fallback to blob' logic in backend API and This keeps API behavior unchanged for clients and preserves backward compatibility.

for eg:
  function getBillingRecord(recordId) {
  record = CosmosDB.get(recordId)
  if (record) return record
  else return BlobStorage.readJsonBlob(recordId)
}

## 2. Code implementation:

### A. Azure Function for Archiving data in Storage Blob (Python)

```python
import datetime
import json
import logging
import azure.functions as func
from azure.cosmos import CosmosClient, PartitionKey
from azure.storage.blob import BlobServiceClient, ContentSettings

# Cosmos DB Configuration
COSMOS_ENDPOINT = "<COSMOS-DB-ENDPOINT>"
COSMOS_KEY = "<COSMOS-DB-KEY>"
DATABASE_NAME = "<DATABASE-NAME>"
CONTAINER_NAME = "<CONTAINER-NAME>"

# Azure Blob Storage Configuration
BLOB_CONNECTION_STRING = "<BLOB-STORAGE-CONNECTION-STRING>"
BLOB_CONTAINER_NAME = "<BLOB-CONTAINER-NAME>"

def main(mytimer: func.TimerRequest) -> None:
    utc_timestamp = datetime.datetime.utcnow()
    logging.info(f"Python timer trigger function ran at {utc_timestamp}")

    # Initialize Cosmos Client
    cosmos_client = CosmosClient(COSMOS_ENDPOINT, COSMOS_KEY)
    database = cosmos_client.get_database_client(DATABASE_NAME)
    container = database.get_container_client(CONTAINER_NAME)

    # Initialize Blob Service Client
    blob_service_client = BlobServiceClient.from_connection_string(BLOB_CONNECTION_STRING)
    blob_container_client = blob_service_client.get_container_client(BLOB_CONTAINER_NAME)

    # Calculate Date Threshold (Older than 3 Months)
    threshold_date = (utc_timestamp - datetime.timedelta(days=90)).isoformat()

    query = f"SELECT * FROM c WHERE c.date < '{threshold_date}' AND (NOT IS_DEFINED(c.archived) OR c.archived = false)"
    items = container.query_items(query=query, enable_cross_partition_query=True)

    for item in items:
        record_id = item['id']
        partition_key = item['partitionKey']  # Adjust based on your schema
        
        # Prepare JSON blob content
        blob_name = f"{record_id}.json"
        blob_data = json.dumps(item)

        # Upload to Blob Storage (Cool Tier)
        blob_client = blob_container_client.get_blob_client(blob_name)
        blob_client.upload_blob(blob_data, overwrite=True, content_settings=ContentSettings(content_type='application/json'), standard_blob_tier="Cool")

        # Mark item as archived in Cosmos DB
        item['archived'] = True
        container.upsert_item(item)

        logging.info(f"Archived Record ID: {record_id} to Blob Storage")

    logging.info("Archival process completed.")
```

### B. Blob Storage Data organtization example:

customer123/date/abcde123.json

## 3. Architectural diagram
![Architecture Diagram](./Architectural%20Diagram.jpg)


## 4. Monitoring Logs recommendations

- Enable Azure Application Insights on Azure Functions and API service.
- Log successes, failures, fallback reads, and latency metrics.
