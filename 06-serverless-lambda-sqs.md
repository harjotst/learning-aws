# Module 06 · Serverless: Lambda Functions and SQS Queues

**Time: about 20 minutes.**
**Before you start:** open CloudShell and check the region is `us-east-1`. Check the orders table from
Module 05 exists:
```bash
aws dynamodb describe-table --table-name lab-orders --query Table.TableStatus --output text
```
It should print `ACTIVE`. If you get `ResourceNotFoundException`, redo Module 05, Part 3, Steps 1-2.

**Where you are:** in Module 04 you ran code on servers you had to launch, wait for, patch, and delete.
In this module you'll run code with **no server at all**, and connect it to the rest of a system through
a **queue**, the most common way AWS applications are wired together.

**What you'll build:**

```
 You (pretending to be a      ┌──────────────────┐      ┌──────────────────┐      ┌──────────────┐
 checkout page)  ──message──▶ │ SQS queue        │ ───▶ │ Lambda function  │ ───▶ │ DynamoDB     │
 aws sqs send-message         │ lab-orders-queue │      │ lab-order-worker │      │ lab-orders   │
                              └────────┬─────────┘      └──────────────────┘      └──────────────┘
                                       │ a message that fails 3 times
                                       ▼
                              ┌──────────────────┐
                              │ lab-orders-dlq   │  "dead-letter queue": bad messages wait here
                              └──────────────────┘
```

Everything in this module costs **nothing** at this scale (Lambda and SQS include free usage every month).

---

## Part 1: Concepts (7 min read)

### 1.1 What "serverless" means

There are still servers, but **you never see or manage them**. You hand AWS your code (or your data,
for S3 and DynamoDB), and AWS decides where it runs, starts more copies when traffic grows, and
**charges only while it's actually working**. You've already used two serverless services: S3 and DynamoDB.

### 1.2 Lambda

**AWS Lambda** runs a **function**, a piece of your code, whenever an **event** happens. Examples of events:
a file uploaded to S3, an HTTP request arriving, a message arriving in a queue, or a schedule ("every day at 9am").

The pieces:

| Piece | What it is | Ours |
|---|---|---|
| **Runtime** | The language environment | Python 3.13 |
| **Handler** | The function Lambda calls, written `file_name.function_name` | `lambda_function.handler` |
| **Event** | The data describing what happened, passed to your handler as its first argument | A batch of SQS messages |
| **Execution role** | The IAM role the function **wears** (just like EC2 in Module 04) | `lab-order-worker-role` |
| **Trigger** | What invokes it | The SQS queue |
| **Memory / timeout** | 128 MB-10 GB of memory (CPU grows with memory); max run time 15 minutes | 128 MB, 5 seconds |
| **Environment variables** | Settings passed to your code | `TABLE_NAME=lab-orders` |

What happens when an event arrives:
1. If there's no idle copy of your function ready, Lambda starts a new one: it loads the runtime, then
   runs your code **outside** the handler (imports, connections). This is a **cold start**, typically
   100 ms to 1 second.
2. It calls your **handler** with the event.
3. It keeps that copy warm for a while and reuses it for later events (a **warm start**, with no setup cost).
4. If many events arrive at once, it runs many copies in parallel. This is called **concurrency** (by default up to 1,000 at a time per region).

**Anything your function prints goes to CloudWatch Logs automatically.** That's where you debug.

**Price:** per request (fractions of a cent per thousand) plus per millisecond of run time multiplied by
memory. A function that isn't being called costs nothing.

### 1.3 Why put a queue in the middle?

Imagine a checkout page that saves the order, charges the card, emails a receipt, and updates the warehouse,
all in the same request. If the email service is slow, checkout is slow. If the warehouse system is down,
checkout **fails**, and you lose the sale.

With a **queue**, checkout just drops a message saying "order O-1001 was placed" and immediately tells the
customer "thanks!". Separate workers pick up messages and do the slow work **at their own pace**, retrying
if something is down. This is called **decoupling**, and it's how almost every serious AWS system is built.

### 1.4 SQS

**SQS (Simple Queue Service)** is a managed queue:
- A **producer** sends **messages** (text up to about 1 MB; typically a small JSON document).
- A **consumer** receives messages, processes them, then **deletes** them to mark them done.
- **Visibility timeout:** when a consumer receives a message, SQS **hides** it from other consumers for N
  seconds. If the consumer deletes it in time, it's done. If the consumer crashes or fails, the message
  **reappears** after the timeout and is retried. That's how SQS makes sure no message is lost.
- **Dead-letter queue (DLQ):** if a message has been received `maxReceiveCount` times without being deleted,
  SQS moves it to a separate queue, the DLQ. Without that, one broken message (a **"poison message"**)
  would be retried forever. **Always configure a DLQ.**
- **Standard queues** (what we use) are nearly unlimited in speed but deliver each message **at least once**:
  occasionally a message arrives **twice**. So your consumer must be **idempotent**, meaning processing
  the same message twice has the same result as processing it once. You'll use the DynamoDB condition from
  Module 05 (Step 6) for exactly this.
- **FIFO queues** guarantee order and exactly-once processing, at lower throughput.

When Lambda is connected to an SQS queue, **Lambda does the receiving and deleting for you**: it polls the
queue, calls your handler with a batch of messages, and deletes them if your handler finishes without an
error. If your handler raises an error, the messages aren't deleted, so they reappear and get retried.

### 1.5 The other "glue" services (you'll meet these in real systems)

| Service | Pattern | Use it when |
|---|---|---|
| **SNS** | Publish/subscribe: one message → many subscribers | One event must trigger several things (email, analytics, warehouse), each via its own SQS queue |
| **EventBridge** | Event bus with routing rules, plus schedules | Route events by their content ("orders over $1000 → fraud check"), react to AWS events, run cron jobs |
| **Step Functions** | Workflow / state machine | A multi-step process with branching, waits, and retries: "charge card → reserve stock → if that fails, refund" |
| **API Gateway** | HTTP front door for Lambda | You want a URL/API that runs a function. You'll use it in Module 08. |
| **Kinesis** | Ordered, replayable data stream | Large volumes of events (clicks, telemetry) that several consumers read, and may re-read |

---

## Part 2: Lab (13 min)

### Step 1: Create the function's execution role (with a deliberate gap)

**Do this:**
```bash
cat > lambda-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
aws iam create-role --role-name lab-order-worker-role \
  --assume-role-policy-document file://lambda-trust-policy.json \
  --query Role.Arn --output text

aws iam attach-role-policy --role-name lab-order-worker-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaSQSQueueExecutionRole
```

**What you should see:** the role's ARN, `arn:aws:iam::123456789012:role/lab-order-worker-role`.

**What just happened:**
- **Trust policy:** only the **Lambda service** may wear this role (compare Module 02, where it was EC2).
- **Permissions:** the AWS-managed policy `AWSLambdaSQSQueueExecutionRole` allows reading and deleting SQS
  messages, and writing logs to CloudWatch.
- **Deliberately missing:** permission to write to DynamoDB. You'll see what that failure looks like in Step 4,
  then fix it. This is the most common Lambda error you'll ever debug.

### Step 2: Write the function code

**Do this** (paste the whole block including `EOF`):
```bash
cat > lambda_function.py <<'EOF'
import json
import os
from decimal import Decimal

import boto3

# Code outside the handler runs once, when a new copy of the function starts
# (a "cold start"). The connection to DynamoDB is then reused by every request.
table = boto3.resource("dynamodb").Table(os.environ["TABLE_NAME"])


def handler(event, context):
    records = event["Records"]
    print(f"Received {len(records)} message(s)")

    for record in records:
        order = json.loads(record["body"])
        print(f"Processing order {order['order_id']} for customer {order['customer_id']}")

        if order["total"] < 0:
            raise ValueError(f"Order {order['order_id']} has a negative total")

        try:
            table.put_item(
                Item={
                    "PK": f"CUSTOMER#{order['customer_id']}",
                    "SK": f"ORDER#{order['date']}#{order['order_id']}",
                    "status": "PENDING",
                    "total": Decimal(str(order["total"])),
                    "items": order.get("items", []),
                },
                ConditionExpression="attribute_not_exists(PK)",
            )
            print(f"Saved order {order['order_id']}")
        except table.meta.client.exceptions.ConditionalCheckFailedException:
            print(f"Order {order['order_id']} was already saved; skipping the duplicate")

    return {"processed": len(records)}
EOF
zip function.zip lambda_function.py
```

**What you should see:** `adding: lambda_function.py (deflated 52%)`.

**What the code does, section by section:**
- `import boto3`: **boto3** is the AWS SDK for Python (Module 00, Part 5). It's pre-installed in Lambda.
  It gets credentials from the execution role automatically; there's no key anywhere.
- `table = boto3.resource("dynamodb").Table(os.environ["TABLE_NAME"])`: connect to the table named in the
  environment variable `TABLE_NAME`. It's **outside** the handler, so it runs once per cold start, not per message.
- `def handler(event, context):`: the function Lambda calls. `event` holds the SQS messages under
  `event["Records"]`. Each record's `body` is the text that was sent to the queue, which we parse as JSON.
- `if order["total"] < 0: raise ValueError(...)`: bad data **raises an error**. The message won't be deleted,
  so SQS retries it, and after 3 tries moves it to the DLQ.
- `table.put_item(... ConditionExpression="attribute_not_exists(PK)")`: save the order in the same key format as
  Module 05, **only if it isn't already there**.
- `except ...ConditionalCheckFailedException:`: if it's already there (a duplicate delivery), log it and move on
  instead of failing. **This makes the function idempotent.**
- `Decimal(str(order["total"]))`: DynamoDB's Python library requires `Decimal` instead of floating-point numbers.
- `zip function.zip lambda_function.py`: Lambda takes code as a **.zip file**.

### Step 3: Create the function

**Do this:**
```bash
aws lambda create-function \
  --function-name lab-order-worker \
  --runtime python3.13 \
  --handler lambda_function.handler \
  --zip-file fileb://function.zip \
  --role arn:aws:iam::$ACCOUNT_ID:role/lab-order-worker-role \
  --timeout 5 \
  --memory-size 128 \
  --environment "Variables={TABLE_NAME=lab-orders}" \
  --query '[FunctionName,Runtime,State]' --output text

aws lambda wait function-active-v2 --function-name lab-order-worker && echo "Function is active"
```

**What you should see:** `lab-order-worker	python3.13	Pending`, then `Function is active`.

**What each option means:**
- `--handler lambda_function.handler`: call the function `handler` in the file `lambda_function.py`.
- `--zip-file fileb://function.zip`: upload the zip. (`fileb://` = read as binary; `file://` = read as text.)
- `--role ...`: the execution role from Step 1.
- `--timeout 5`: stop it if it runs longer than 5 seconds.
- `--environment "Variables={TABLE_NAME=lab-orders}"`: sets the environment variable the code reads.

**If it fails:** `The role defined for the function cannot be assumed by Lambda` means the new role hasn't
spread through IAM yet (Module 02, Step 11). Wait 10 seconds and run the `create-function` command again.

### Step 4: Invoke it by hand, see it fail, and read the error

Before connecting the queue, test the function by calling it yourself with a **test event**: a hand-written
imitation of what SQS will send.

**Do this:**
```bash
cat > test-event.json <<'EOF'
{
  "Records": [
    {
      "messageId": "test-1",
      "body": "{\"order_id\": \"T-0001\", \"customer_id\": \"42\", \"date\": \"2026-09-27\", \"total\": 45.99, \"items\": [\"headphones\"]}"
    }
  ]
}
EOF
aws lambda invoke \
  --function-name lab-order-worker \
  --cli-binary-format raw-in-base64-out \
  --payload file://test-event.json \
  response.json
cat response.json; echo
```

**What you should see:**
```json
{
    "StatusCode": 200,
    "FunctionError": "Unhandled",
    "ExecutedVersion": "$LATEST"
}
{"errorMessage": "An error occurred (AccessDeniedException) when calling the PutItem operation: User: arn:aws:sts::123456789012:assumed-role/lab-order-worker-role/lab-order-worker is not authorized to perform: dynamodb:PutItem on resource: arn:aws:dynamodb:us-east-1:123456789012:table/lab-orders because no identity-based policy allows the dynamodb:PutItem action", "errorType": "ClientError", ...}
```

**What just happened:**
- `aws lambda invoke` ran the function once with `test-event.json` as the event and wrote the function's
  return value (or error) to `response.json`.
- `--cli-binary-format raw-in-base64-out` tells the CLI the payload file is plain JSON. (Without it, CLI v2
  expects base64-encoded input. Just always include it.)
- The message `body` is a JSON document **inside a string**, which is why its quotes are escaped as `\"`. That's
  exactly how SQS delivers it.
- `StatusCode: 200` means Lambda ran your code. `FunctionError: Unhandled` means **your code raised an error**.
- Read the error the way Module 02, Step 11 taught: **who** (`assumed-role/lab-order-worker-role`), **action**
  (`dynamodb:PutItem`), **resource** (the `lab-orders` table), **why** (`no identity-based policy allows`).
  The fix is obvious from the message.

### Step 5: Fix the permission and invoke again

**Do this:**
```bash
cat > write-orders-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "dynamodb:PutItem",
      "Resource": "arn:aws:dynamodb:us-east-1:$ACCOUNT_ID:table/lab-orders"
    }
  ]
}
EOF
aws iam put-role-policy --role-name lab-order-worker-role \
  --policy-name write-orders-table --policy-document file://write-orders-policy.json
sleep 10

aws lambda invoke --function-name lab-order-worker \
  --cli-binary-format raw-in-base64-out --payload file://test-event.json response.json
cat response.json; echo
```

**What you should see:**
```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
{"processed": 1}
```
No `FunctionError` this time.

**Invoke the exact same event once more:**
```bash
aws lambda invoke --function-name lab-order-worker \
  --cli-binary-format raw-in-base64-out --payload file://test-event.json response.json > /dev/null
cat response.json; echo
```
**What you should see:** `{"processed": 1}` again. It didn't fail, even though the order already exists. Step 6 shows what happened inside.

### Step 6: Read the function's logs

**Do this:**
```bash
aws logs tail /aws/lambda/lab-order-worker --since 15m --format short
```

**What you should see** (your IDs and numbers will differ):
```
2026-09-27T15:50:01 INIT_START Runtime Version: python:3.13...
2026-09-27T15:50:01 START RequestId: 1b2c... Version: $LATEST
2026-09-27T15:50:01 Received 1 message(s)
2026-09-27T15:50:01 Processing order T-0001 for customer 42
2026-09-27T15:50:02 [ERROR] ClientError: An error occurred (AccessDeniedException) ...
2026-09-27T15:50:02 END RequestId: 1b2c...
2026-09-27T15:50:02 REPORT RequestId: 1b2c... Duration: 612.33 ms Billed Duration: 613 ms Memory Size: 128 MB Max Memory Used: 82 MB Init Duration: 498.10 ms
...
2026-09-27T15:51:15 Processing order T-0001 for customer 42
2026-09-27T15:51:15 Saved order T-0001
...
2026-09-27T15:51:40 Processing order T-0001 for customer 42
2026-09-27T15:51:40 Order T-0001 was already saved; skipping the duplicate
```

**How to read it:**
- Each invocation is wrapped in `START` ... `END` ... `REPORT`. Your `print()` lines are in between.
- `REPORT` shows **Duration** (how long your code ran), **Billed Duration** (what you pay for), and **Max Memory Used**.
- `Init Duration` appears **only on cold starts**: the time spent starting a new copy and running the code
  outside the handler. Later invocations that reuse the warm copy don't have it.
- The last invocation shows **idempotency** working: the duplicate was noticed and skipped safely.

Check the order is really in the table:
```bash
aws dynamodb get-item --table-name lab-orders \
  --key '{"PK": {"S": "CUSTOMER#42"}, "SK": {"S": "ORDER#2026-09-27#T-0001"}}' \
  --query 'Item.[status.S,total.N]' --output text
```
**What you should see:** `PENDING	45.99`

### Step 7: Create the dead-letter queue and the main queue

**Do this:**
```bash
DLQ_URL=$(aws sqs create-queue --queue-name lab-orders-dlq \
  --attributes MessageRetentionPeriod=1209600 \
  --query QueueUrl --output text)
save DLQ_URL
DLQ_ARN=$(aws sqs get-queue-attributes --queue-url $DLQ_URL \
  --attribute-names QueueArn --query Attributes.QueueArn --output text)
save DLQ_ARN

cat > queue-attributes.json <<EOF
{
  "VisibilityTimeout": "10",
  "RedrivePolicy": "{\"deadLetterTargetArn\":\"$DLQ_ARN\",\"maxReceiveCount\":\"3\"}"
}
EOF
QUEUE_URL=$(aws sqs create-queue --queue-name lab-orders-queue \
  --attributes file://queue-attributes.json \
  --query QueueUrl --output text)
save QUEUE_URL
QUEUE_ARN=$(aws sqs get-queue-attributes --queue-url $QUEUE_URL \
  --attribute-names QueueArn --query Attributes.QueueArn --output text)
save QUEUE_ARN
```

**What you should see:** four `saved:` lines. Queue URLs look like `https://sqs.us-east-1.amazonaws.com/123456789012/lab-orders-queue`.

**What just happened:**
- The **DLQ** keeps messages for 14 days (`1209600` seconds), long enough for someone to investigate.
- The **main queue**:
  - `VisibilityTimeout: 10`: a received message is hidden for 10 seconds. It must be at least the function's
    timeout (5 s), otherwise a message could reappear while it's still being processed.
  - `RedrivePolicy`: after **3** failed receives, move the message to the DLQ. Its value is a JSON document
    written *inside a string*, which is why its inner quotes are escaped.
- SQS identifies queues by **URL** for sending/receiving and by **ARN** for permissions and connections, so we saved both.

### Step 8: Connect the queue to the function

**Do this:**
```bash
MAPPING_UUID=$(aws lambda create-event-source-mapping \
  --function-name lab-order-worker \
  --event-source-arn $QUEUE_ARN \
  --batch-size 1 \
  --query UUID --output text)
save MAPPING_UUID

until [ "$(aws lambda get-event-source-mapping --uuid $MAPPING_UUID --query State --output text)" = "Enabled" ]; do
  echo "Connecting queue to function..."; sleep 5
done; echo "Queue is connected to the function"
```

**What you should see:** a few `Connecting...` lines, then `Queue is connected to the function` (usually within 30-60 seconds).

**What just happened:** an **event source mapping** tells Lambda: "poll this queue, and call this function with
the messages." `--batch-size 1` means one message per invocation, which makes the behaviour easy to follow
(the default is 10, which is more efficient in real use). The `until ... done` loop checks every 5 seconds
until the connection's state is `Enabled`.

**If it fails:** `The provided execution role does not have permissions to call ReceiveMessage on SQS` means the
`AWSLambdaSQSQueueExecutionRole` policy isn't attached. Redo the `attach-role-policy` line from Step 1.

### Step 9: Send orders, including a duplicate and a bad one

**Do this:**
```bash
GOOD='{"order_id": "O-1001", "customer_id": "42", "date": "2026-09-27", "total": 89.00, "items": ["webcam"]}'
BAD='{"order_id": "O-6666", "customer_id": "13", "date": "2026-09-27", "total": -5.00, "items": ["???"]}'

aws sqs send-message --queue-url $QUEUE_URL --message-body "$GOOD" --query MessageId --output text
aws sqs send-message --queue-url $QUEUE_URL --message-body "$GOOD" --query MessageId --output text
aws sqs send-message --queue-url $QUEUE_URL --message-body "$BAD"  --query MessageId --output text
```

**What you should see:** three message IDs (long random strings).

**What just happened:** you acted as the checkout page. You sent a good order, the **same** good order again
(simulating a duplicate delivery), and a bad order with a negative total.

### Step 10: Watch the system handle all three

**Do this:**
```bash
aws logs tail /aws/lambda/lab-order-worker --since 2m --follow --format short
```
Watch for about 40 seconds, then press **Ctrl + C** to stop following.

**What you should see:**
```
... Processing order O-1001 for customer 42
... Saved order O-1001
... Processing order O-1001 for customer 42
... Order O-1001 was already saved; skipping the duplicate
... Processing order O-6666 for customer 13
... [ERROR] ValueError: Order O-6666 has a negative total
      (about 10 seconds later)
... Processing order O-6666 for customer 13
... [ERROR] ValueError: Order O-6666 has a negative total
      (about 10 seconds later)
... Processing order O-6666 for customer 13
... [ERROR] ValueError: Order O-6666 has a negative total
```

**What just happened:**
1. The good order was saved.
2. The duplicate was detected and skipped, so there's no double order.
3. The bad order failed. The message wasn't deleted, so after the 10-second visibility timeout it reappeared and
   was retried. After the **third** failure, SQS moved it to the dead-letter queue.

**Check the dead-letter queue:**
```bash
aws sqs get-queue-attributes --queue-url $DLQ_URL \
  --attribute-names ApproximateNumberOfMessages --query Attributes --output text
aws sqs receive-message --queue-url $DLQ_URL --query 'Messages[0].Body' --output text
```
**What you should see:** `1`, then the bad order's JSON. If the count is `0`, wait 20 seconds and try again.

**Check customer 42's orders:**
```bash
aws dynamodb query --table-name lab-orders \
  --key-condition-expression "PK = :c" \
  --expression-attribute-values '{":c": {"S": "CUSTOMER#42"}}' \
  --no-scan-index-forward \
  --query 'Items[].[SK.S, total.N]' --output table
```
**What you should see:** `O-1001` and `T-0001` at the top (both dated 2026-09-27), followed by the two orders from Module 05. Customer 13 has nothing, because the bad order never made it in.

**This is a complete, production-shaped pipeline:** the producer never waited for processing, a duplicate caused no
harm, and a poison message was set aside for a human instead of blocking everything or disappearing.

### Step 11: See it in the console

1. Search `Lambda` → **Functions** → `lab-order-worker`:
   - The **diagram** at the top shows the SQS trigger on the left.
   - **Code** tab: your code, editable in the browser.
   - **Monitor** tab: graphs of invocations, errors, and duration. **View CloudWatch logs** opens the logs you tailed.
2. Search `SQS` → `lab-orders-dlq` → **Send and receive messages** → **Poll for messages**: you can inspect the
   bad message. In real life, after fixing the bug you'd use **Start DLQ redrive** on this queue to send its messages
   back to the main queue for another try.

---

## Part 3: Cleanup (2 min)

**Do this:**
```bash
aws lambda delete-event-source-mapping --uuid $MAPPING_UUID --query State --output text
aws lambda delete-function --function-name lab-order-worker && echo "Function deleted"
aws sqs delete-queue --queue-url $QUEUE_URL && echo "Queue deleted"
aws sqs delete-queue --queue-url $DLQ_URL && echo "DLQ deleted"
aws logs delete-log-group --log-group-name /aws/lambda/lab-order-worker && echo "Logs deleted"
aws iam delete-role-policy --role-name lab-order-worker-role --policy-name write-orders-table
aws iam detach-role-policy --role-name lab-order-worker-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaSQSQueueExecutionRole
aws iam delete-role --role-name lab-order-worker-role && echo "Role deleted"
aws dynamodb delete-table --table-name lab-orders --query TableDescription.TableStatus --output text
```

**What you should see:** `Deleting`, then the four `deleted` messages, then `Role deleted`, then `DELETING`.

**What just happened:** you removed everything from Modules 05 and 06. The Module 08 capstone creates its own table.

---

## Checkpoint

1. What's a cold start, and which part of your code runs during it?
2. Your function returns `FunctionError: Unhandled` with `AccessDeniedException ... dynamodb:PutItem`. What do you change?
3. Why did the bad order get processed three times?
4. What would have happened to the bad order without a dead-letter queue?
5. Why must a consumer of a standard SQS queue be idempotent, and how did yours achieve it?
6. Why is the checkout page better off sending a message to a queue than calling the worker directly?

<details><summary>Answers</summary>

1. When no warm copy exists, Lambda starts a new one and runs everything outside the handler (imports, the DynamoDB connection) before calling the handler. It shows up as `Init Duration` in the REPORT log line.
2. Add `dynamodb:PutItem` on the table's ARN to the function's **execution role** permission policy.
3. The handler raised an error, so the message wasn't deleted. After the visibility timeout it reappeared and was retried, until it reached `maxReceiveCount` (3).
4. It would have been retried until the queue's retention period expired (4 days by default), wasting invocations and cluttering logs, and then silently vanished.
5. Standard queues deliver at least once, so duplicates happen. The function writes with `attribute_not_exists(PK)` and treats "already exists" as success.
6. Checkout stays fast and keeps working even if the worker or its downstream systems are slow or down. Messages wait in the queue and get retried.
</details>

**Where you are now:** you've built servers, networks, storage, databases, and serverless pipelines, all by typing
commands one at a time, and deleted them the same way. Next, you'll learn how professionals avoid doing that by hand,
and how to watch over a running system.

**Next: [Module 07 · Infrastructure as Code & Operations](07-iac-and-operations.md)**
