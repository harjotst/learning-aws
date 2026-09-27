# Module 05 · Storage & Databases: S3 and DynamoDB

**Time: about 20 minutes.**
**Before you start:** open CloudShell, check the region is `us-east-1`, and run `echo $BUCKET`. It
should print `lab-website-<account-id>`. If it's empty, redo Module 02, Steps 2-3.

**Where you are:** your web servers read a file from S3, but you've only used S3 as a simple file
store. Real applications also need a **database** for data that changes all the time: orders, users,
messages. This module covers both, and the database table you create here is where Module 06's
code will save orders.

**What you'll do:**
1. Use the S3 features that matter in real work: prefixes, encryption, blocking public access,
   temporary share links, bucket policies, version history, and automatic archiving.
2. Create a **DynamoDB** table of customer orders, then read and write it the right way (and see
   the wrong way).

---

## Part 1: Concepts (8 min read)

### 1.1 Three kinds of storage

| Kind | How you use it | AWS service | Everyday comparison |
|---|---|---|---|
| **Object storage** | Put/get whole files by name over HTTP. Can't edit part of a file in place. Effectively unlimited. | **S3** | A giant online drive (like Dropbox) |
| **Block storage** | A raw disk attached to one server; the operating system formats it. | **EBS** (the disk your EC2 instances had) | The hard drive inside a computer |
| **File storage** | A shared folder many servers mount at the same time. | **EFS** | A shared network drive in an office |

### 1.2 S3 in detail

- A **bucket** holds **objects** (files). Each object has a **key**, its full name, e.g.
  `reports/2026/q3.pdf`. Objects can be up to 5 TB.
- **There are no real folders.** `reports/2026/q3.pdf` is one key that happens to contain slashes.
  The console and CLI *display* the part before a `/` as a folder, called a **prefix**.
- **Durability:** S3 stores each object on multiple devices across at least three AZs. It's designed
  so that if you store 10 million objects, you'd expect to lose one about every 10,000 years
  ("11 nines" of durability, 99.999999999%).
- **Security by default:** new buckets have **Block Public Access** turned on (nothing in them can be
  made public) and **encryption at rest** turned on (files are encrypted on AWS's disks).
- **How to share files safely:** don't make the bucket public. Instead use:
  - **Presigned URLs**: a link that includes a signature made with *your* credentials and an expiry
    time. Anyone holding the link can download (or upload) that one object until it expires. This is
    how websites let users upload and download files directly to S3.
  - **CloudFront** (AWS's CDN, a network of caching servers worldwide) in front of the bucket, to
    serve a website fast and securely.
- **Bucket policy:** a **resource-based policy** (Module 02, 1.4) attached to the bucket, saying who may
  do what *to this bucket*. It has a `Principal` field.
- **Versioning:** when on, overwriting or deleting a file keeps the old version, so you can recover from mistakes.
- **Storage classes:** the same data can be stored at different price levels, depending on how often you read it:

| Class | Use for | Trade-off |
|---|---|---|
| `STANDARD` | Frequently read data | Most expensive storage, no retrieval fee |
| `INTELLIGENT_TIERING` | Unknown access patterns | AWS moves objects between tiers for you |
| `STANDARD_IA` ("infrequent access") | Read about once a month | Cheaper storage, fee per GB read |
| `GLACIER_IR` / `GLACIER` / `DEEP_ARCHIVE` | Archives, backups, compliance | Very cheap storage; reading can take minutes to hours |

- **Lifecycle rules** move or delete objects automatically based on age, e.g. "after 30 days move to
  IA, after 90 days to Glacier, after a year delete."
- **Event notifications:** S3 can trigger other services (such as a Lambda function) whenever a file is uploaded.

### 1.3 What a database is, and the two main kinds

A **database** stores data that your application reads and changes constantly, and lets you find
items quickly without reading everything.

**Relational databases** (PostgreSQL, MySQL, and others) store data in **tables** with fixed **columns**,
like spreadsheets that link to each other. You query them with **SQL**, e.g.
`SELECT * FROM orders WHERE customer_id = 42`. They're great for complex questions ("total sales per
region last quarter, joined with customer info"). On AWS:
- **RDS** runs PostgreSQL, MySQL, MariaDB, SQL Server, Oracle, or Db2 for you. AWS handles installation,
  patching, and backups.
  - **Multi-AZ**: a standby copy in another AZ that takes over automatically in about a minute if the main
    one fails. This is for **availability**.
  - **Read replicas**: extra copies you can send read queries to. This is for **handling more reads**.
- **Aurora**: AWS's own PostgreSQL/MySQL-compatible database, built for higher performance and availability.

**NoSQL key-value databases** give up flexible queries in exchange for speed at any scale. You fetch
items by their **key**. On AWS that's **DynamoDB**:
- **Serverless**: no servers, versions, or connections to manage. You create a table and use it.
  Response times stay in single-digit milliseconds whether the table holds 10 items or 10 billion.
- **You design the table around the questions you'll ask it**, because you can only look things up
  efficiently by key.

### 1.4 DynamoDB in detail

- A **table** holds **items** (like rows). Each item is a set of **attributes** (like columns), but
  items don't need to have the same attributes. Max item size is 400 KB.
- Every item has a **primary key**, made of:
  - a **partition key** (required): DynamoDB uses it to decide where to store the item. All items with
    the same partition key are stored together.
  - a **sort key** (optional): orders items that share a partition key, and lets you ask for ranges.
- **Query** finds items with **one partition key value** (optionally narrowed by sort key). It's fast
  and cheap because it only reads what you asked for.
- **Scan** reads **every item in the table**. It's fine for a tiny table and slow and expensive for a big
  one. Avoid it in application code.
- A **Global Secondary Index (GSI)** is a second copy of the table, automatically maintained, with a
  **different key**, so you can Query by something else.
- **Capacity:** **On-demand** (pay per request, no planning; what we use) or **provisioned** (you reserve
  reads/writes per second; cheaper for steady traffic).

The table you'll build stores orders so you can answer **"show me customer 42's orders, newest first"**:

```
Partition key (PK)   Sort key (SK)                  status     total   items
CUSTOMER#42          ORDER#2026-09-01#A17           SHIPPED    30.00   [keyboard]
CUSTOMER#42          ORDER#2026-09-20#B03           PENDING    12.50   [mouse pad]
CUSTOMER#77          ORDER#2026-08-11#C99           PENDING    99.00   [monitor]
CUSTOMER#77          ORDER#2026-09-25#D42           SHIPPED    15.00   [cable]
```
- Putting the date at the start of the sort key means the sort order is date order.
- The `CUSTOMER#` and `ORDER#` prefixes are a common convention that makes keys self-explanatory.
- To also answer **"show me all PENDING orders"**, you'll add a GSI keyed on `status`.

### 1.5 Which database to choose

| Choose | When |
|---|---|
| **RDS / Aurora** (relational) | You need flexible queries, joins, and reports; your access patterns aren't known yet; or the app already uses SQL |
| **DynamoDB** | You know exactly how you'll look data up, you need consistent speed at any scale, or you're building serverless |
| **ElastiCache** (Valkey/Redis) | You need a super-fast in-memory cache in front of another database, or session storage |

---

## Part 2: S3 lab (8 min)

### Step 1: Look at an object's details

**Do this:**
```bash
aws s3 ls s3://$BUCKET
aws s3api head-object --bucket $BUCKET --key index.html \
  --query '{Size:ContentLength,Type:ContentType,Encryption:ServerSideEncryption,Class:StorageClass}'
```

**What you should see:**
```
2026-09-27 15:04:11        262 index.html
{
    "Size": 262,
    "Type": "text/html",
    "Encryption": "AES256",
    "Class": null
}
```

**What just happened:** `aws s3` is the simple, file-like command set; `aws s3api` gives you every
low-level S3 operation. `Encryption: AES256` means it's encrypted at rest automatically. `Class: null`
means `STANDARD` (S3 leaves it blank for the default).

### Step 2: See that "folders" are just prefixes

**Do this:**
```bash
echo "Q3 revenue: 1,000,000" > q3.txt
aws s3 cp q3.txt s3://$BUCKET/reports/2026/q3.txt
aws s3 ls s3://$BUCKET/
aws s3 ls s3://$BUCKET/ --recursive
```

**What you should see:**
```
                           PRE reports/
2026-09-27 15:04:11        262 index.html

2026-09-27 15:04:11        262 index.html
2026-09-27 15:20:02         22 reports/2026/q3.txt
```

**What just happened:** you never created a `reports` folder. The first listing shows `PRE reports/`
("prefix") because the CLI groups keys by the text before the `/`. With `--recursive` you see the real
key: `reports/2026/q3.txt`.

### Step 3: Confirm the bucket is private

**Do this:**
```bash
aws s3api get-public-access-block --bucket $BUCKET
curl -s https://$BUCKET.s3.amazonaws.com/index.html
```

**What you should see:**
```json
{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": true,
        "IgnorePublicAcls": true,
        "BlockPublicPolicy": true,
        "RestrictPublicBuckets": true
    }
}
```
followed by an XML error containing `<Code>AccessDenied</Code>`.

**What just happened:** all four Block Public Access settings are on. `curl` requested the file's public
URL without any credentials, and S3 refused. Only identities with permission (like you, or your role in
Module 04) can read it.

### Step 4: Share a file temporarily with a presigned URL

**Do this:**
```bash
URL=$(aws s3 presign s3://$BUCKET/reports/2026/q3.txt --expires-in 300)
echo "$URL"
curl -s "$URL"
```

**What you should see:** a long URL (containing `X-Amz-Signature=...` and `X-Amz-Expires=300`), then
`Q3 revenue: 1,000,000`.

**What just happened:** `presign` created a link that carries *your* permission, signed with your
credentials, valid for 300 seconds (5 minutes). `curl` has no AWS credentials, but the link itself proves
permission. You can paste it into any browser and it works until it expires. **Always quote `"$URL"`**, because
it contains `&` characters that the shell would otherwise misinterpret.

Now watch one expire:
```bash
SHORT_URL=$(aws s3 presign s3://$BUCKET/reports/2026/q3.txt --expires-in 5)
sleep 8
curl -s "$SHORT_URL"
```
**What you should see:** XML with `<Code>AccessDenied</Code>` and `<Message>Request has expired</Message>`.

### Step 5: Add a bucket policy that requires encrypted connections

A common company rule: "never allow unencrypted (plain `http://`) access to our buckets." You enforce it
with a bucket policy.

**Do this:**
```bash
cat > bucket-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedConnections",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::$BUCKET", "arn:aws:s3:::$BUCKET/*"],
      "Condition": { "Bool": { "aws:SecureTransport": "false" } }
    }
  ]
}
EOF
aws s3api put-bucket-policy --bucket $BUCKET --policy file://bucket-policy.json

URL=$(aws s3 presign s3://$BUCKET/reports/2026/q3.txt --expires-in 300)
echo "HTTPS:"; curl -s "$URL"; echo
echo "HTTP:";  curl -s "${URL/https:/http:}" | grep -o '<Code>.*</Code>'
```

**What you should see:**
```
HTTPS:
Q3 revenue: 1,000,000
HTTP:
<Code>AccessDenied</Code>
```

**What just happened:**
- This policy has a `Principal` (`"*"`, meaning *anyone*) because it's attached to the **resource** (the
  bucket), not to a user. Compare to the identity policies in Module 02, which had no Principal.
- It **denies** every S3 action when the condition `aws:SecureTransport` is `false`, i.e. when the request
  didn't use HTTPS.
- `${URL/https:/http:}` is a bash trick that swaps `https:` for `http:` in the variable. The exact same valid,
  signed link was refused over plain HTTP, because **an explicit Deny beats the Allow** carried by the signature
  (Module 02, 1.5).

### Step 6: Turn on versioning and recover a deleted file

**Do this: enable versioning, then overwrite the page**
```bash
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled

sed 's/Hello from AWS!/Hello again, version 2!/' index.html > index-v2.html
aws s3 cp index-v2.html s3://$BUCKET/index.html

aws s3api list-object-versions --bucket $BUCKET --prefix index.html \
  --query 'Versions[].[VersionId,IsLatest,LastModified]' --output table
```

**What you should see:**
```
+------------------------------------+--------+-----------------------------+
|  3HL4kqtJlcpXroDTDmJ.rmSpXd3dIbrHY |  True  |  2026-09-27T15:31:02+00:00  |
|  null                              |  False |  2026-09-27T15:04:11+00:00  |
+------------------------------------+--------+-----------------------------+
```

**What just happened:** you now have two versions. The new one is `IsLatest: True`. The old one has the
version ID **`null`**, because it was uploaded *before* versioning was turned on. Every version uploaded from now
on gets a real ID.

**Do this: read the old version**
```bash
aws s3api get-object --bucket $BUCKET --key index.html --version-id null old.html > /dev/null
grep h1 old.html
aws s3 cp s3://$BUCKET/index.html - | grep h1
```
**What you should see:** `<h1>Hello from AWS!</h1>` (old version), then `<h1>Hello again, version 2!</h1>` (current).
(`> /dev/null` hides the command's JSON output; the file content goes to `old.html`.)

**Do this: "delete" the file, then bring it back**
```bash
aws s3 rm s3://$BUCKET/index.html
aws s3 ls s3://$BUCKET/
aws s3api list-object-versions --bucket $BUCKET --prefix index.html \
  --query '{versions: Versions[].VersionId, deleteMarkers: DeleteMarkers[].VersionId}'
```
**What you should see:** `index.html` is gone from the listing, but the versions output still shows both
versions, plus one **delete marker**.

**What just happened:** with versioning on, a delete doesn't erase anything. It adds a **delete marker**,
a placeholder that says "this file is deleted" and hides the file. Remove the marker and the file returns:

```bash
MARKER=$(aws s3api list-object-versions --bucket $BUCKET --prefix index.html \
  --query 'DeleteMarkers[0].VersionId' --output text)
aws s3api delete-object --bucket $BUCKET --key index.html --version-id $MARKER > /dev/null
aws s3 ls s3://$BUCKET/
```
**What you should see:** `index.html` is back.

### Step 7: Add lifecycle rules (automatic archiving and cleanup)

**Do this:**
```bash
cat > lifecycle.json <<'EOF'
{
  "Rules": [
    {
      "ID": "archive-old-reports",
      "Filter": { "Prefix": "reports/" },
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 365 }
    },
    {
      "ID": "delete-old-versions",
      "Filter": { "Prefix": "" },
      "Status": "Enabled",
      "NoncurrentVersionExpiration": { "NoncurrentDays": 30 }
    }
  ]
}
EOF
aws s3api put-bucket-lifecycle-configuration --bucket $BUCKET --lifecycle-configuration file://lifecycle.json
aws s3api get-bucket-lifecycle-configuration --bucket $BUCKET --query 'Rules[].[ID,Status]' --output table
```

**What you should see:** a table with `archive-old-reports | Enabled` and `delete-old-versions | Enabled`.

**What just happened:** you told S3:
1. Objects whose key starts with `reports/`: move to `STANDARD_IA` after 30 days, to `GLACIER` after 90, delete after 365.
2. All objects: delete **old versions** 30 days after they're replaced, so versioning doesn't pile up cost forever.

S3 applies these rules on its own, once a day. You'll never see them act during this course, but this is
exactly how companies keep storage bills under control.

### Step 8: See it in the console

Search `S3` → click your bucket. Look at:
- **Objects** tab: `index.html` and the `reports/` "folder". Toggle **Show versions** to see the versions and delete marker history.
- **Properties** tab: Bucket Versioning **Enabled**, Default encryption **SSE-S3**.
- **Permissions** tab: Block public access **On**, and your bucket policy.
- **Management** tab: your two lifecycle rules.

---

## Part 3: DynamoDB lab (8 min)

### Step 1: Create the orders table

**Do this:**
```bash
aws dynamodb create-table \
  --table-name lab-orders \
  --attribute-definitions \
      AttributeName=PK,AttributeType=S \
      AttributeName=SK,AttributeType=S \
      AttributeName=status,AttributeType=S \
  --key-schema \
      AttributeName=PK,KeyType=HASH \
      AttributeName=SK,KeyType=RANGE \
  --global-secondary-indexes '[{
      "IndexName": "status-index",
      "KeySchema": [
        {"AttributeName": "status", "KeyType": "HASH"},
        {"AttributeName": "SK", "KeyType": "RANGE"}
      ],
      "Projection": {"ProjectionType": "ALL"}
    }]' \
  --billing-mode PAY_PER_REQUEST \
  --query TableDescription.TableStatus --output text

aws dynamodb wait table-exists --table-name lab-orders && echo "Table is ready"
```

**What you should see:** `CREATING`, then after about 10-20 seconds, `Table is ready`.

**What each part means:**
- `--attribute-definitions`: declares the type of **only the attributes used in keys** (`S` = string).
  Other attributes don't need declaring. DynamoDB is schemaless apart from keys.
- `--key-schema`: `PK` is the **partition key** (`HASH` is DynamoDB's old name for it) and `SK` is the
  **sort key** (`RANGE`).
- `--global-secondary-indexes`: an index named `status-index` with `status` as its partition key and `SK`
  as its sort key. `"ProjectionType": "ALL"` copies every attribute into the index.
- `--billing-mode PAY_PER_REQUEST`: on-demand pricing. An idle table costs nothing.

### Step 2: Add four orders

DynamoDB's CLI uses a **typed JSON** format. Each value is wrapped in an object that says its type:
`{"S": "text"}` for a string, `{"N": "12.50"}` for a number (always written as a string), `{"L": [...]}` for a list.

**Do this:**
```bash
aws dynamodb put-item --table-name lab-orders --item '{
  "PK": {"S": "CUSTOMER#42"}, "SK": {"S": "ORDER#2026-09-01#A17"},
  "status": {"S": "SHIPPED"}, "total": {"N": "30.00"}, "items": {"L": [{"S": "keyboard"}]}
}'
aws dynamodb put-item --table-name lab-orders --item '{
  "PK": {"S": "CUSTOMER#42"}, "SK": {"S": "ORDER#2026-09-20#B03"},
  "status": {"S": "PENDING"}, "total": {"N": "12.50"}, "items": {"L": [{"S": "mouse pad"}]}
}'
aws dynamodb put-item --table-name lab-orders --item '{
  "PK": {"S": "CUSTOMER#77"}, "SK": {"S": "ORDER#2026-08-11#C99"},
  "status": {"S": "PENDING"}, "total": {"N": "99.00"}, "items": {"L": [{"S": "monitor"}]}
}'
aws dynamodb put-item --table-name lab-orders --item '{
  "PK": {"S": "CUSTOMER#77"}, "SK": {"S": "ORDER#2026-09-25#D42"},
  "status": {"S": "SHIPPED"}, "total": {"N": "15.00"}, "items": {"L": [{"S": "cable"}]}
}'
echo "4 orders written"
```

**What you should see:** `4 orders written` (`put-item` prints nothing on success).

**What just happened:** you wrote four items. Note that `put-item` **replaces** any existing item with the
same `PK` + `SK`. You'll see how to prevent that in Step 6.

### Step 3: Get one exact item

**Do this:**
```bash
aws dynamodb get-item --table-name lab-orders \
  --key '{"PK": {"S": "CUSTOMER#42"}, "SK": {"S": "ORDER#2026-09-01#A17"}}'
```

**What you should see:** the full item in typed JSON, under `"Item"`.

**What just happened:** `get-item` needs the **complete primary key** (both PK and SK) and returns exactly one
item. It's the fastest, cheapest operation.

### Step 4: Query a customer's orders, newest first

**Do this:**
```bash
aws dynamodb query --table-name lab-orders \
  --key-condition-expression "PK = :customer" \
  --expression-attribute-values '{":customer": {"S": "CUSTOMER#42"}}' \
  --no-scan-index-forward \
  --query 'Items[].[SK.S, status.S, total.N]' --output table
```

**What you should see:**
```
+-----------------------+----------+--------+
|  ORDER#2026-09-20#B03 |  PENDING |  12.50 |
|  ORDER#2026-09-01#A17 |  SHIPPED |  30.00 |
+-----------------------+----------+--------+
```

**What just happened:**
- `--key-condition-expression "PK = :customer"`: "items whose partition key equals the value `:customer`".
  Words starting with `:` are **placeholders**, and their real values come from `--expression-attribute-values`.
- `--no-scan-index-forward`: sort by the sort key **descending**. Since the sort key starts with the date,
  that's newest first.
- Only customer 42's partition was read. Customer 77's orders were never touched, however many there are.

Now narrow by sort key, to get only September's orders:
```bash
aws dynamodb query --table-name lab-orders \
  --key-condition-expression "PK = :customer AND begins_with(SK, :month)" \
  --expression-attribute-values '{":customer": {"S": "CUSTOMER#77"}, ":month": {"S": "ORDER#2026-09"}}' \
  --query 'Items[].[SK.S, total.N]' --output table
```
**What you should see:** only `ORDER#2026-09-25#D42 | 15.00`. Customer 77's August order is excluded.

### Step 5: Query the index for all PENDING orders (and meet reserved words)

**Do this first (it will fail on purpose):**
```bash
aws dynamodb query --table-name lab-orders --index-name status-index \
  --key-condition-expression "status = :s" \
  --expression-attribute-values '{":s": {"S": "PENDING"}}'
```

**What you should see:**
```
An error occurred (ValidationException) when calling the Query operation: Invalid KeyConditionExpression:
Attribute name is a reserved keyword; reserved keyword: status
```

**What just happened:** DynamoDB has several hundred **reserved words** (`status`, `name`, `date`, `count`, ...)
that can't appear directly in expressions. The fix is a **name placeholder** starting with `#`:

**Do this:**
```bash
aws dynamodb query --table-name lab-orders --index-name status-index \
  --key-condition-expression "#st = :s" \
  --expression-attribute-names '{"#st": "status"}' \
  --expression-attribute-values '{":s": {"S": "PENDING"}}' \
  --query 'Items[].[PK.S, SK.S, total.N]' --output table
```
**What you should see:** the two PENDING orders (B03 for customer 42 and C99 for customer 77).

**What just happened:** `#st` stands for the attribute *name* `status` (from `--expression-attribute-names`), and
`:s` stands for the *value* `PENDING`. You queried the **GSI**, which is keyed by status, so this is also a fast
Query and not a Scan.

### Step 6: Update an item, and protect against overwrites

**Do this: mark order B03 as shipped**
```bash
aws dynamodb update-item --table-name lab-orders \
  --key '{"PK": {"S": "CUSTOMER#42"}, "SK": {"S": "ORDER#2026-09-20#B03"}}' \
  --update-expression "SET #st = :new" \
  --expression-attribute-names '{"#st": "status"}' \
  --expression-attribute-values '{":new": {"S": "SHIPPED"}}' \
  --return-values ALL_NEW --query 'Attributes.status.S' --output text
```
**What you should see:** `SHIPPED`. `update-item` changes only the attributes you mention.

**Do this: try to create order A17 again, but only if it doesn't already exist**
```bash
aws dynamodb put-item --table-name lab-orders --item '{
  "PK": {"S": "CUSTOMER#42"}, "SK": {"S": "ORDER#2026-09-01#A17"},
  "status": {"S": "PENDING"}, "total": {"N": "0"}
}' --condition-expression "attribute_not_exists(PK)"
```
**What you should see:**
```
An error occurred (ConditionalCheckFailedException) when calling the PutItem operation: The conditional request failed
```

**What just happened:** a **condition expression** makes the write happen *only if* the condition is true.
`attribute_not_exists(PK)` means "only if no item with this key exists yet." A17 exists, so DynamoDB refused,
and the real order wasn't overwritten with a total of 0. **Remember this.** In Module 06 you'll learn that
messages can be delivered twice, and this is how you make sure processing the same order twice does no harm.

### Step 7: Scan (and why you avoid it)

**Do this:**
```bash
aws dynamodb scan --table-name lab-orders --query '[Count, ScannedCount]' --output text
```
**What you should see:** `4	4`.

**What just happened:** Scan read every item in the table. With 4 items that's nothing. With 50 million items it
would read (and charge you for) all 50 million, every time. Use Query or GetItem in application code. Keep Scan
for one-off admin tasks.

### Step 8: See it in the console

Search `DynamoDB` → **Tables** → `lab-orders`:
- **Explore table items** (button top-right): browse and edit items. Try switching between **Scan** and **Query** in the form.
- **Indexes** tab: `status-index`.
- **Monitor** tab: read and write activity graphs.

---

## Part 4: RDS, briefly (optional reading, no lab)

We don't create a relational database in this course because it takes 10+ minutes to start. Here's what the
console's **RDS → Create database** page asks, and what each choice means, so you'll recognize it:

| Setting | What to choose, and why |
|---|---|
| **Engine** | PostgreSQL or MySQL are the usual choices. Aurora if you need more scale. |
| **Template** | *Free tier* for learning, *Production* for real use (turns on Multi-AZ and more). |
| **Availability** | *Multi-AZ DB instance* for production: a standby in another AZ with automatic failover. |
| **Credentials** | Choose *Managed in AWS Secrets Manager*, so the password is stored and rotated for you, never written in code. |
| **Instance class** | Size, like EC2 types, e.g. `db.t4g.micro`. |
| **Connectivity → VPC / subnet group** | Put it in your VPC's **private** subnets (Module 03). |
| **Public access** | **No.** Databases should never be reachable from the internet. |
| **Security group** | One that allows port 5432 (PostgreSQL) **only from your app servers' security group**, the same trick as `lab-web-sg` in Module 03. |
| **Backups** | Automated backups with point-in-time restore, kept 7-35 days. |

---

## Part 5: Cleanup

### Delete the bucket (you're done with it)

A versioned bucket must have **every version and delete marker** removed before it can be deleted.

**Do this:**
```bash
aws s3api delete-objects --bucket $BUCKET --delete "$(aws s3api list-object-versions --bucket $BUCKET \
  --query '{Objects: [Versions, DeleteMarkers][].{Key: Key, VersionId: VersionId}}' --output json)" \
  --query 'length(Deleted)'
aws s3 rb s3://$BUCKET && echo "Bucket deleted"
rm -f q3.txt index-v2.html old.html
```

**What you should see:** a number (how many versions were deleted), then `remove_bucket: lab-website-...` and `Bucket deleted`.

**What just happened:** the inner command lists every version and delete marker as JSON in the exact format
`delete-objects` expects, and the outer command deletes them all in one request. Then `rb` ("remove bucket") works.

**If it fails:** `BucketNotEmpty` means some versions remained (e.g. more than 1,000). Run the same two
commands again, or use the console: S3 → select the bucket → **Empty** → type `permanently delete` → then **Delete**.

### Keep the DynamoDB table

**Don't delete `lab-orders`.** Module 06 writes orders into it. An idle on-demand table costs nothing.

---

## Checkpoint

1. How would you let a user download a private file from S3 without making anything public?
2. Your S3 file was overwritten by mistake. Versioning is on. How do you get the old one back?
3. Why is `status = :s` rejected by DynamoDB, and how do you fix it?
4. Your table's key is `PK = CUSTOMER#...`. A new feature needs "all orders over $50 across all customers." What happens if you just Scan, and what would you do instead?
5. What does `attribute_not_exists(PK)` protect you from?
6. Your app needs flexible reports that join customers, orders, and products. RDS or DynamoDB?

<details><summary>Answers</summary>

1. Generate a **presigned URL** with a short expiry.
2. List the object's versions and download (or copy back) the previous version by its version ID.
3. `status` is a reserved word. Use a name placeholder: `#st = :s` with `--expression-attribute-names '{"#st": "status"}'`.
4. Scan reads the whole table, which gets slow and expensive as it grows. Add a GSI designed for the question, or, for analytics, export the data to S3 and query it with Athena.
5. Accidentally overwriting an existing item, for example processing the same order twice.
6. RDS (relational). Joins and ad-hoc reporting are what SQL is for.
</details>

**Where you are now:** you have a DynamoDB table ready for orders. Next, you'll write code that runs
without any server at all, triggered by messages in a queue, and saves orders into this table.

**Next: [Module 06 · Serverless: Lambda & SQS](06-serverless-lambda-sqs.md)**
