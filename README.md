
Goal: Reduce Cosmos DB costs by archiving old data to cheaper storage (like Azure Blob or Azure Data Lake), while preserving access latency via a fallback mechanism.
ChatGPT
You said:
Cost Optimization Challenge: Managing Billing Records in Azure Serverless Architecture
We have a serverless architecture in Azure, where one of our services stores billing records in Azure Cosmos DB. The system is read-heavy, but records older than three months are rarely accessed.

Over the past few years, the database size has significantly grown, leading to increased costs. We need an efficient way to reduce costs while maintaining data availability.

Current System Constraints
Record Size: Each billing record can be as large as 300 KB.

Total Records: The database currently holds over 2 million records.

Access Latency: When an old record is requested, it should still be served, with a response time in the order of seconds.

Solution Requirements
Please propose a detailed solution to optimize costs while ensuring the following
Simplicity & Ease of Implementation – The solution should be straightforward to deploy and maintain.

No Data Loss & No Downtime – The transition should be seamless, without losing any records or requiring2 service downtime.

No Changes to API Contracts – The existing read/write APIs for billing records must remain unchanged

Bonus Points
Include an architecture diagram illustrating your proposed solution.

Provide pseudocode, commands, or scripts for implementing core logic (such as data archival, retrieval, and cost optimization st rategies).

Please share the solution in a Github repo. If you are using chatGPT or other LLMs to solve please share the conversation.
Do's and Don'ts
1. Please do not add your answers here as a comment, create a public Github repo and share the link with us over the chat (again not here)
2. It is completely okay if you use chatGPT or other tools but we want to access the conversation to understand the nuances of the solution. The better you interact with the AI tools the better your chances
3. Please go beyond the obvious and think about where the system could break. Consider the different problems we might face, how we would tackle them, and how we would fix them if we were to run your solution in a production environment that could impact thousands of users.
4. It is okay to submit the answer twice within a week
ChatGPT said:
Great challenge — here’s how we’ll tackle it:

✅ What You Need to Do Now
To proceed, you should:

Create a public GitHub repository.

Include the following in the repo:

Architecture diagram (showing live + archived data access)

Core logic scripts (in Python or PowerShell) for:

Archiving old records from Cosmos DB to cheaper storage

Accessing archived data seamlessly

Pseudocode or module layout (showing logic integration)

README explaining the solution, migration plan, and considerations

Paste this conversation into the repo so reviewers can assess your reasoning and LLM usage.

✅ What I’ll Do for You
I’ll now prepare:

The solution architecture (with logic flow + diagram)

A breakdown of cost optimization strategy

Sample pseudocode/scripts

How you can integrate without changing existing APIs

Once you're ready, you can create the GitHub repo and copy the materials there.

🧠 High-Level Solution Strategy
Goal: Reduce Cosmos DB costs by archiving old data to cheaper storage (like Azure Blob or Azure Data Lake), while preserving access latency via a fallback mechanism.
🔧 Architecture Overview
🔄 Tiered Storage Strategy
Active Records (< 90 days) → Stored in Cosmos DB (Fast reads)

Archived Records (≥ 90 days) → Moved to Azure Blob Storage (Hot/Cold tier) in Parquet/JSONL format

API calls check Cosmos DB first → fallback to archived storage if not found

✅ Key Benefits
No API changes required (can wrap the fallback logic)

Simple to implement

Significant cost savings (Blob storage is much cheaper than Cosmos DB RU/s + storage)

Zero downtime via scheduled background migration

🗺️ Architecture Diagram
I will generate a clean architecture diagram. Please hold on.

🛠️ Core Logic Breakdown
🔁 Archival Script (Run Daily/Weekly via Azure Function or Logic App)
Query Cosmos DB for records older than 90 days

Write them to blob storage (in folders partitioned by year/month)

Delete them from Cosmos DB (optional: only if backup is successful)

Sample Python Script (using azure-cosmos and azure-storage-blob)
from azure.cosmos import CosmosClient, PartitionKey
from azure.storage.blob import BlobServiceClient
from datetime import datetime, timedelta
import json

COSMOS_URL = "<cosmos-endpoint>"
COSMOS_KEY = "<cosmos-key>"
DATABASE = "BillingDB"
CONTAINER = "Records"

BLOB_CONN_STR = "<blob-connection-string>"
BLOB_CONTAINER = "archived-billing-records"

cutoff_date = datetime.utcnow() - timedelta(days=90)

client = CosmosClient(COSMOS_URL, credential=COSMOS_KEY)
db = client.get_database_client(DATABASE)
container = db.get_container_client(CONTAINER)

query = f"SELECT * FROM c WHERE c.timestamp < '{cutoff_date.isoformat()}'"
old_records = list(container.query_items(query=query, enable_cross_partition_query=True))

# Write to blob
blob_service_client = BlobServiceClient.from_connection_string(BLOB_CONN_STR)
blob_client = blob_service_client.get_blob_client(
    container=BLOB_CONTAINER,
    blob=f"archive/{cutoff_date.strftime('%Y/%m/%d')}/billing_records.json"
)

blob_client.upload_blob(json.dumps(old_records), overwrite=True)

# Optional: delete from Cosmos DB
for record in old_records:
    container.delete_item(item=record['id'], partition_key=record['partitionKey'])
------------------------------
Fallback Read Logic
Updated Read Flow (In API layer or behind Cosmos DB SDK proxy)
Try fetching from Cosmos DB.

If not found → Query Blob Storage.

Return unified result.

Pseudocode:
def get_billing_record(record_id):
    try:
        return cosmos_container.read_item(record_id, partition_key)
    except:
        # Not found in Cosmos DB, check archive
        blob_path = get_archive_path(record_id)
        record = read_from_blob(blob_path, record_id)
        return record

------------------------------------------
API Compatibility (No Contract Change)
You can wrap this logic in a shared data access layer (DAL) used by your APIs, so the endpoint itself remains unchanged. Your codebase continues calling get_billing_record(record_id) without worrying where it lives.

Production Considerations
| Concern                         | Solution                                                         |
| ------------------------------- | ---------------------------------------------------------------- |
| Data consistency                | Only delete after archive confirmation                           |
| Record access pattern may shift | Keep some buffer (e.g., last 3–4 months) in Cosmos DB            |
| Archived data format            | Use Parquet or JSONL for scalable queries                        |
| Cost tracking                   | Use Azure Cost Management for Cosmos + Blob                      |
| Blob indexing                   | Optionally use Azure Data Lake + ADX for faster archive querying |

Deployment Flow
Set up Blob Storage container

Write and schedule archival script as Azure Function or Logic App (daily)

Update read layer to check blob if Cosmos fails

Monitor and validate both storage and access logs
