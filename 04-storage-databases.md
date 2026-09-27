# 04 · Storage & Databases (25 min)

## Mental model

Pick storage by **how data is accessed**, not by what it is:

| Access pattern | Service |
|---|---|
| Files/blobs by key over HTTP, effectively unlimited | **S3** (object storage) |
| A disk attached to one EC2 instance | **EBS** (block storage) |
| A shared filesystem mounted by many instances/containers | **EFS** (NFS) / FSx (Windows, Lustre, NetApp, ZFS) |
| Relational data, joins, transactions, SQL | **RDS** / **Aurora** |
| Key-value/document lookups at any scale, single-digit ms | **DynamoDB** |
| Cache / sessions / leaderboards, sub-ms | **ElastiCache** (Valkey/Redis OSS, Memcached) |
| Analytics over large data | **Redshift** (warehouse), **Athena** (SQL over files in S3) |
| Full-text search / log analytics | **OpenSearch** |

---

## S3 (Simple Storage Service)

The oldest AWS service, and it shows up in nearly every architecture.

- **Bucket**: a container with a globally unique name, living in one region.
- **Object**: a file up to 5 TB, addressed by a **key** such as `photos/2026/cat.jpg`. There are
  no real folders; the `/` is just part of the key (the console fakes folders from prefixes).
- **Durability 99.999999999% (11 nines)**, since data is copied across ≥3 AZs. **Strongly
  consistent**: read-after-write returns the new data.
- **Security defaults (since 2023):** new buckets have **Block Public Access on**, ACLs
  disabled, and **encryption at rest (SSE-S3)** on. Leave Block Public Access on. To serve files
  publicly, put **CloudFront** in front (with Origin Access Control) or use **presigned URLs**.
- **Presigned URL**: a time-limited URL signed with your credentials, so someone without AWS access can GET or PUT one object. This is the standard way to do browser uploads and downloads.
- **Versioning**: keeps every version of an object, which protects against overwrites and deletes. Pair it with lifecycle rules so old versions don't pile up.
- **Storage classes** (same API, different price/retrieval trade-offs):
  - `STANDARD`: hot data
  - `INTELLIGENT_TIERING`: AWS moves objects between tiers based on access. A good default when you don't know the pattern.
  - `STANDARD_IA` / `ONEZONE_IA`: infrequent access, cheaper storage, retrieval fee
  - `GLACIER_IR` / `GLACIER_FLEXIBLE_RETRIEVAL` / `GLACIER_DEEP_ARCHIVE`: archives. Flexible and Deep Archive restores take minutes to hours.
- **Lifecycle rules**: e.g. "move to IA after 30 days, Glacier after 90, delete after 365."
- **Event notifications**: object created → Lambda / SQS / SNS / EventBridge. This is the starting point of many pipelines.
- **Static website hosting**: S3 + CloudFront is the standard way to host a SPA.
- **Performance**: scales automatically, about 3,500 writes and 5,500 reads per second *per prefix*.

## EBS & EFS (quick)
- **EBS**: lives in **one AZ**, attaches to one instance (with some exceptions). Use `gp3` by
  default. `io2` is for high-IOPS databases. **Snapshots** go to S3 and can be copied across
  regions. That's how you back up and move volumes.
- **EFS**: NFS that many instances or containers in many AZs can mount at once. It grows automatically and costs more per GB than EBS.

---

## Relational: RDS & Aurora

**RDS** is a managed database: AWS handles provisioning, patching, backups, and failover.
Engines: PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, Db2.

- **Multi-AZ**: a synchronous standby in another AZ with automatic failover (~1-2 min) that
  keeps the same DNS endpoint. This is for **availability**, not read scaling.
- **Read replicas**: asynchronous copies you read from. This is for **read scaling** (and they can be cross-region).
- **Automated backups** with point-in-time restore (up to 35 days), plus manual snapshots.
- Put it in **private DB subnets**, allow access only from the app's security group, and keep credentials in **Secrets Manager**.
- **RDS Proxy**: pools connections. It's essential when many Lambdas connect to a relational database.

**Aurora** is AWS's cloud-native MySQL/PostgreSQL-compatible engine. Storage is replicated
6 ways across 3 AZs, it supports up to 15 low-lag replicas and faster failover, and
**Aurora Serverless v2** scales capacity automatically. (Aurora DSQL is a newer distributed,
serverless Postgres-compatible option for multi-region active-active.)

---

## DynamoDB

A fully managed NoSQL key-value/document database. No servers or connections to manage, and
consistent single-digit-ms latency whether the table holds 1 KB or 100 TB.

**The catch: you design the table around your queries, up front.** You can only query
efficiently by key.

- **Partition key (PK)** (required): hashed to decide which partition stores the item.
- **Sort key (SK)** (optional): orders items within a partition, and allows range queries (`begins_with`, `between`, `>`).
- **Primary key** = PK, or PK + SK. It must be unique.
- **Item** max size **400 KB**. Items are schemaless apart from the key attributes.
- **Query** reads items from *one partition* (fast, cheap). **Scan** reads the *whole table* (slow and expensive, so avoid it in hot paths).
- **GSI (Global Secondary Index)**: an alternative PK/SK for a different access pattern. It's
  eventually consistent and effectively another copy of the table.
- **Capacity**: **On-Demand** (pay per request, the default choice) or **Provisioned** (+ auto scaling, cheaper at steady load).
- Extras: **TTL** (auto-expire items), **Streams** (change feed → Lambda), transactions, **Global Tables** (multi-region, multi-active).

Example: an orders table that serves "get a customer's orders, newest first":
```
PK                 SK                          attributes...
CUSTOMER#42        ORDER#2026-09-01#A17        total=30.00 status=SHIPPED
CUSTOMER#42        ORDER#2026-09-20#B03        total=12.50 status=PENDING
CUSTOMER#77        ORDER#2026-08-11#C99        ...
```
`Query PK = "CUSTOMER#42" AND begins_with(SK, "ORDER#2026-09")`, `ScanIndexForward=false` → newest first.
For "all PENDING orders", add a GSI with PK=`status`, SK=`createdAt`.

### RDS or DynamoDB?
- **RDS/Aurora** when you need ad-hoc queries, joins, complex reporting, or an existing SQL app, or when the access patterns aren't known yet.
- **DynamoDB** when the access patterns are known, scale or latency matters, you're serverless, or you want zero ops.

---

## Lab: S3 + DynamoDB (10 min)

```bash
export AWS_REGION=us-east-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET=storage-lab-$ACCOUNT_ID

# --- S3: versioning + presigned URL ---
aws s3 mb s3://$BUCKET
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled

echo "v1" > note.txt && aws s3 cp note.txt s3://$BUCKET/note.txt
echo "v2" > note.txt && aws s3 cp note.txt s3://$BUCKET/note.txt
aws s3api list-object-versions --bucket $BUCKET --prefix note.txt \
  --query 'Versions[].{Id:VersionId,Latest:IsLatest}' --output table   # both versions kept

aws s3 rm s3://$BUCKET/note.txt       # "delete" only adds a delete marker
aws s3api list-object-versions --bucket $BUCKET --prefix note.txt \
  --query 'DeleteMarkers[].VersionId' --output text   # remove this marker to "undelete"

echo "secret report" > report.txt && aws s3 cp report.txt s3://$BUCKET/report.txt
URL=$(aws s3 presign s3://$BUCKET/report.txt --expires-in 120)
curl -s "$URL"                         # ✅ works for anyone with the URL for 2 minutes
curl -s "https://$BUCKET.s3.amazonaws.com/report.txt" | head -c 200   # ❌ AccessDenied (not public)

# --- DynamoDB: design by access pattern ---
aws dynamodb create-table --table-name lab-orders \
  --attribute-definitions AttributeName=PK,AttributeType=S AttributeName=SK,AttributeType=S \
  --key-schema AttributeName=PK,KeyType=HASH AttributeName=SK,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST
aws dynamodb wait table-exists --table-name lab-orders

for item in 'CUSTOMER#42|ORDER#2026-09-01#A17|30.00' 'CUSTOMER#42|ORDER#2026-09-20#B03|12.50' 'CUSTOMER#77|ORDER#2026-08-11#C99|99.00'; do
  IFS='|' read pk sk total <<< "$item"
  aws dynamodb put-item --table-name lab-orders \
    --item "{\"PK\":{\"S\":\"$pk\"},\"SK\":{\"S\":\"$sk\"},\"total\":{\"N\":\"$total\"}}"
done

# Customer 42's orders, newest first. Only touches one partition.
aws dynamodb query --table-name lab-orders \
  --key-condition-expression "PK = :pk AND begins_with(SK, :prefix)" \
  --expression-attribute-values '{":pk":{"S":"CUSTOMER#42"},":prefix":{"S":"ORDER#"}}' \
  --no-scan-index-forward --query 'Items[].[SK.S,total.N]' --output table
```

### Cleanup
```bash
aws dynamodb delete-table --table-name lab-orders
# a versioned bucket needs every version deleted first:
aws s3api delete-objects --bucket $BUCKET --delete "$(aws s3api list-object-versions --bucket $BUCKET \
  --query '{Objects: [Versions, DeleteMarkers][].{Key:Key,VersionId:VersionId}}' --output json)"
aws s3 rb s3://$BUCKET
rm -f note.txt report.txt
```

## Check yourself

1. Users upload profile photos from the browser. How do you do this without proxying bytes through your server or making the bucket public?
2. RDS Multi-AZ vs read replica: which one fixes "the DB is slow under read load"?
3. You need "all orders over $100 across all customers" from the table above. What's wrong with it, and what are your options?
4. 7-year compliance archive that's almost never read. Storage class?
5. Two EC2 instances in different AZs need the same files. EBS or EFS?

<details><summary>Answers</summary>

1. Your backend generates a **presigned PUT URL** and the browser uploads straight to S3.
2. A read replica. Multi-AZ standbys don't serve reads (on standard RDS). They're for failover.
3. The table is keyed by customer, so this needs a Scan. Options: add a GSI designed for it, or if it's analytics, export to S3 and query with Athena. This is why you design DynamoDB around access patterns.
4. `GLACIER_DEEP_ARCHIVE` (cheapest, hours to restore), via a lifecycle rule.
5. EFS. An EBS volume is tied to one AZ and generally one instance.
</details>

**Next → [05 Serverless & Integration](05-serverless-integration.md)**
