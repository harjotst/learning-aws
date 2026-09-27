# 05 · Serverless & Integration (25 min)

## Mental model

Services in a well-built AWS system **don't call each other directly when they don't have to**.
They pass messages and events through managed "glue" services. That glue is what lets each part
scale, fail, and deploy on its own.

```
Tightly coupled:   Checkout ──sync call──▶ Email service   (email down ⇒ checkout fails)
Decoupled:         Checkout ──▶ [queue/event] ──▶ Email service   (email down ⇒ messages wait)
```

## The glue services

| Service | Pattern | Think of it as | Use when |
|---|---|---|---|
| **SQS** | Queue (point-to-point) | A to-do list that workers pull from | Buffer work, absorb spikes, retry failures |
| **SNS** | Pub/sub (push, fan-out) | A megaphone | One event → many subscribers (SQS, Lambda, email, SMS, HTTP) |
| **EventBridge** | Event bus + routing rules | A smart router | Route events by *content* between services, SaaS, and AWS itself; cron schedules |
| **Kinesis Data Streams** | Ordered, replayable stream | A log/tape (like Kafka) | High-volume ordered data, multiple readers, replay. (**MSK** = managed Kafka) |
| **Step Functions** | Workflow / state machine | A flowchart that runs | Multi-step processes with branching, retries, waits, human approval |

### SQS in depth (you'll use it constantly)
- **Standard** queue: nearly unlimited throughput, **at-least-once** delivery, best-effort order.
  → Your consumers must be **idempotent** (safe to process the same message twice).
- **FIFO** queue: exactly-once processing and strict ordering *within a message group*, with lower throughput.
- **Visibility timeout**: once a consumer receives a message it's hidden for N seconds. If the
  consumer doesn't **delete** it in time, it reappears and gets retried. Set it longer than your
  processing time (for Lambda, ≥ the function timeout; AWS recommends 6×).
- **Dead-letter queue (DLQ)**: after `maxReceiveCount` failed attempts, the message moves to the DLQ
  so a "poison" message can't block processing forever. **Always configure one** and alarm on its depth.
- **Long polling** (`WaitTimeSeconds=20`): fewer empty responses, lower cost.
- Keep messages small (a few hundred KB at most). For big payloads, store them in S3 and send the key.

### Fan-out: SNS + SQS
```
                      ┌──▶ SQS: email-queue    ──▶ email worker
Order placed ──▶ SNS ─┼──▶ SQS: invoice-queue  ──▶ invoice Lambda
                      └──▶ SQS: analytics-queue──▶ analytics
```
Each consumer gets its own copy and its own queue, so a slow or broken consumer only affects
itself. EventBridge can do the same with content-based routing rules (`detail.amount > 100 → fraud-check`).

### EventBridge specifics
- Events are JSON with `source`, `detail-type`, `detail`. **Rules** match patterns and send to **targets**.
- AWS services publish events to the default bus automatically (e.g. "EC2 instance stopped", "S3 object created").
- **EventBridge Scheduler**: cron/rate schedules and one-off timers. This is the modern "cron job" on AWS.
- **Pipes**: point-to-point source → filter → enrich → target, with no glue code.

### API Gateway
The front door for HTTP APIs that are backed by Lambda (or anything else).
- **HTTP API**: cheaper and simpler. The default choice for Lambda proxies, JWT auth, and CORS.
- **REST API**: more features (API keys and usage plans, request validation, caching, WAF integration, private APIs).
- **WebSocket API**: for real-time two-way connections.
- The integration timeout is 29 seconds by default. Anything longer should be async (return 202 and process via a queue).
- **Lambda Function URLs** give one function a simple HTTPS endpoint without API Gateway.

### A typical serverless architecture
```
Browser ─▶ CloudFront ─▶ S3 (static SPA)
   │
   └─▶ API Gateway ─▶ Lambda ─▶ DynamoDB
          (Cognito JWT auth)  │
                              └─▶ EventBridge ─▶ Lambda (send email via SES)
                                             └─▶ SQS ─▶ Lambda (generate PDF → S3)
```
**Cognito** handles user sign-up and sign-in and issues the JWTs that API Gateway validates.

## Lab: Lambda + SQS with a dead-letter queue (12 min)

You'll build a worker that processes queue messages, then watch a poison message get retried and land in the DLQ.

```bash
export AWS_REGION=us-east-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# 1) Execution role: Lambda can assume it; it can read SQS + write logs
aws iam create-role --role-name lab-worker-role --assume-role-policy-document '{
  "Version":"2012-10-17","Statement":[{"Effect":"Allow",
  "Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name lab-worker-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaSQSQueueExecutionRole

# 2) The function
cat > worker.py <<'EOF'
import json

def handler(event, context):
    for record in event["Records"]:          # SQS delivers a batch of records
        body = json.loads(record["body"])
        print(f"processing order {body['order_id']}")
        if body.get("poison"):
            raise ValueError("cannot process this message")   # message will be retried
    return {"ok": True}
EOF
zip -q worker.zip worker.py
sleep 10   # let the new role propagate
aws lambda create-function --function-name lab-worker --runtime python3.13 \
  --handler worker.handler --zip-file fileb://worker.zip \
  --role arn:aws:iam::$ACCOUNT_ID:role/lab-worker-role --timeout 5

# 3) Queues: main queue redrives to a DLQ after 3 failed receives
DLQ_URL=$(aws sqs create-queue --queue-name lab-dlq --query QueueUrl --output text)
DLQ_ARN=$(aws sqs get-queue-attributes --queue-url $DLQ_URL --attribute-names QueueArn --query Attributes.QueueArn --output text)
cat > attrs.json <<EOF
{ "VisibilityTimeout": "10",
  "RedrivePolicy": "{\"deadLetterTargetArn\":\"$DLQ_ARN\",\"maxReceiveCount\":\"3\"}" }
EOF
Q_URL=$(aws sqs create-queue --queue-name lab-orders --attributes file://attrs.json --query QueueUrl --output text)
Q_ARN=$(aws sqs get-queue-attributes --queue-url $Q_URL --attribute-names QueueArn --query Attributes.QueueArn --output text)

# 4) Wire the queue to the function (Lambda polls SQS for you)
aws lambda create-event-source-mapping --function-name lab-worker \
  --event-source-arn $Q_ARN --batch-size 1

# 5) Send a good message and a poison one
aws sqs send-message --queue-url $Q_URL --message-body '{"order_id": "A17"}'
aws sqs send-message --queue-url $Q_URL --message-body '{"order_id": "BAD1", "poison": true}'

# 6) Watch: A17 processed once; BAD1 fails 3 times (~10 s apart), then goes to the DLQ
aws logs tail /aws/lambda/lab-worker --follow        # Ctrl-C after ~1 minute
aws sqs get-queue-attributes --queue-url $DLQ_URL --attribute-names ApproximateNumberOfMessages
```

**What you just saw:** automatic retries driven by the visibility timeout, isolation of a poison
message, and logs in CloudWatch without any setup. In production you'd put a CloudWatch alarm on
the DLQ depth, and you can "redrive" DLQ messages back to the main queue after fixing the bug
(SQS console → DLQ → *Start DLQ redrive*).

### Cleanup
```bash
UUID=$(aws lambda list-event-source-mappings --function-name lab-worker --query 'EventSourceMappings[0].UUID' --output text)
aws lambda delete-event-source-mapping --uuid $UUID
aws lambda delete-function --function-name lab-worker
aws sqs delete-queue --queue-url $Q_URL
aws sqs delete-queue --queue-url $DLQ_URL
aws iam detach-role-policy --role-name lab-worker-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaSQSQueueExecutionRole
aws iam delete-role --role-name lab-worker-role
aws logs delete-log-group --log-group-name /aws/lambda/lab-worker
rm -f worker.py worker.zip attrs.json
```

## Check yourself

1. Your SQS consumer sometimes processes the same order twice. Bug in SQS?
2. A signup event must trigger a welcome email, a CRM sync, and an analytics record. Design it.
3. An API call kicks off a 5-minute video transcode. Why not do it inside the API's Lambda?
4. You need to replay the last 24 hours of clickstream events into a new consumer. SQS or Kinesis?
5. Order workflow: charge card → reserve stock → if stock fails, refund card → notify. Which service?

<details><summary>Answers</summary>

1. No. Standard queues are at-least-once, and a message whose processing outlives the visibility timeout reappears. Make the consumer idempotent (e.g. a DynamoDB conditional write on `order_id`), or use FIFO with deduplication.
2. SNS topic (or EventBridge rule) → three SQS queues → three consumers. Each one can fail and retry independently.
3. API Gateway times out at 29 s, and the client shouldn't wait anyway. Return `202 Accepted`, put a job on SQS or start a Step Functions execution, and let the client poll or get notified.
4. Kinesis (or MSK). It keeps records for a retention period and many consumers can read them independently. SQS deletes a message once it's consumed.
5. Step Functions. It handles orchestration, retries, and compensation (the saga pattern).
</details>

**Next → [06 IaC, Observability, Security, Cost](06-iac-ops-security-cost.md)**
