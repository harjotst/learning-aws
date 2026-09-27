# Module 08 · Capstone: A Complete Application from One File

**Time: about 15-20 minutes.**
**Before you start:** open CloudShell, check the region is `us-east-1`, and have your email open.

**Where you are:** you've learned each piece separately. Now you'll deploy a complete, working
**notes API**, a small web service that stores and returns notes, using a single CloudFormation
template (Module 07) that combines almost everything from the course:

```
 curl / browser
      │  HTTPS: GET /notes, POST /notes, GET /notes/{id}, DELETE /notes/{id}
      ▼
 ┌──────────────────────┐     ┌───────────────────────────┐     ┌──────────────────────┐
 │ API Gateway          │ ──▶ │ Lambda: NotesFunction     │ ──▶ │ DynamoDB: NotesTable │
 │ (HTTP API)           │     │ Python, wears an IAM role │     │ partition key: id    │
 └──────────────────────┘     │ that can only touch       │     └──────────────────────┘
                              │ NotesTable                │
                              └─────────────┬─────────────┘
                                            │ prints logs            errors metric
                                            ▼                             │
                              ┌───────────────────────────┐     ┌─────────▼────────────┐
                              │ CloudWatch Logs           │     │ CloudWatch Alarm     │──▶ SNS ──▶ your email
                              │ (kept 7 days)             │     │ "any error in 1 min" │
                              └───────────────────────────┘     └──────────────────────┘
```

**Cost:** effectively $0. Everything here is serverless and bills per request.

---

## Part 1: The one new service: API Gateway (2 min read)

**API Gateway** gives your Lambda function a public **HTTPS URL**. When a request arrives, it turns the
HTTP request (method, path, headers, body) into an **event**, invokes your function with it, and turns the
function's return value (`statusCode`, `headers`, `body`) back into an HTTP response.

There are three flavours: **HTTP API** (simple and cheap; what we use), **REST API** (more features, such as API keys,
request validation, and caching), and **WebSocket API** (live two-way connections). An HTTP API created with a
`Target` (as in our template) automatically gets a **default route** that sends *every* request to the function.
The function then looks at the method and path itself to decide what to do.

API Gateway is a *service* calling your function, not a user or role. So the function needs a **resource-based
policy** (Module 02, 1.4) saying "API Gateway may invoke me". That's the `ApiInvokePermission` resource below.

---

## Part 2: Create the template file (3 min)

**Do this:** paste this **entire** block into CloudShell, from `cat` down to and including the final `EOF` line.
(It's the same as [`template.yaml`](template.yaml) in this folder.)

```bash
mkdir -p ~/labs/capstone && cd ~/labs/capstone
cat > template.yaml <<'EOF'
AWSTemplateFormatVersion: "2010-09-09"
Description: >
  Capstone: a serverless notes API.
  HTTP API (API Gateway) -> Lambda (Python) -> DynamoDB, with logs, least-privilege IAM,
  and a CloudWatch alarm on Lambda errors.

Parameters:
  AlarmEmail:
    Type: String
    Default: ""
    Description: Optional. Email to notify when the function errors (you must confirm the subscription email).

Conditions:
  HasAlarmEmail: !Not [!Equals [!Ref AlarmEmail, ""]]

Resources:
  # ---------- Data ----------
  NotesTable:
    Type: AWS::DynamoDB::Table
    Properties:
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH

  # ---------- Identity: the role the function runs as ----------
  NotesFunctionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:          # trust policy: who may assume this role
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal: { Service: lambda.amazonaws.com }
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - !Sub arn:${AWS::Partition}:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:                          # permission policy: least privilege, one table
        - PolicyName: notes-table-access
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action: [dynamodb:GetItem, dynamodb:PutItem, dynamodb:DeleteItem, dynamodb:Scan]
                Resource: !GetAtt NotesTable.Arn

  # ---------- Compute ----------
  NotesLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      RetentionInDays: 7                 # the default is "never expire"

  NotesFunction:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.13
      Architectures: [arm64]             # Graviton: cheaper per ms
      Handler: index.handler
      MemorySize: 256
      Timeout: 10
      Role: !GetAtt NotesFunctionRole.Arn
      LoggingConfig:
        LogGroup: !Ref NotesLogGroup
      Environment:
        Variables:
          TABLE_NAME: !Ref NotesTable
      Code:
        ZipFile: |
          import base64, json, os, time, uuid
          from decimal import Decimal
          import boto3

          # Runs once per execution environment (cold start), then reused
          table = boto3.resource("dynamodb").Table(os.environ["TABLE_NAME"])

          def respond(status, body=None):
              return {
                  "statusCode": status,
                  "headers": {"content-type": "application/json"},
                  "body": "" if body is None else json.dumps(
                      body, default=lambda o: int(o) if isinstance(o, Decimal) else str(o)),
              }

          def handler(event, context):
              method = event.get("requestContext", {}).get("http", {}).get("method") or event.get("httpMethod")
              path = (event.get("rawPath") or event.get("path") or "/").rstrip("/") or "/"
              parts = path.strip("/").split("/")
              print(json.dumps({"level": "INFO", "method": method, "path": path}))

              if parts == ["boom"]:
                  raise RuntimeError("intentional failure to trigger the alarm")
              if parts[0] != "notes" or len(parts) > 2:
                  return respond(404, {"error": "try GET/POST /notes or GET/DELETE /notes/{id}"})

              if len(parts) == 1:
                  if method == "GET":
                      # Scan is fine for a demo table; see module 04 for why not at scale
                      return respond(200, table.scan(Limit=100)["Items"])
                  if method == "POST":
                      raw = event.get("body") or "{}"
                      if event.get("isBase64Encoded"):
                          raw = base64.b64decode(raw).decode()
                      text = json.loads(raw).get("text", "")
                      item = {"id": uuid.uuid4().hex[:8], "text": text, "createdAt": int(time.time())}
                      table.put_item(Item=item)
                      return respond(201, item)
              else:
                  key = {"id": parts[1]}
                  if method == "GET":
                      item = table.get_item(Key=key).get("Item")
                      return respond(200, item) if item else respond(404, {"error": "not found"})
                  if method == "DELETE":
                      table.delete_item(Key=key)
                      return respond(204)
              return respond(405, {"error": "method not allowed"})

  # ---------- Front door ----------
  HttpApi:
    Type: AWS::ApiGatewayV2::Api
    Properties:
      Name: !Sub ${AWS::StackName}-api
      ProtocolType: HTTP
      Target: !GetAtt NotesFunction.Arn  # "quick create": default route + stage -> this Lambda

  ApiInvokePermission:                   # resource-based policy on the function
    Type: AWS::Lambda::Permission
    Properties:
      Action: lambda:InvokeFunction
      FunctionName: !Ref NotesFunction
      Principal: apigateway.amazonaws.com
      SourceArn: !Sub arn:${AWS::Partition}:execute-api:${AWS::Region}:${AWS::AccountId}:${HttpApi}/*

  # ---------- Observability ----------
  AlarmTopic:
    Type: AWS::SNS::Topic
    Condition: HasAlarmEmail
    Properties:
      Subscription:
        - Protocol: email
          Endpoint: !Ref AlarmEmail

  ErrorsAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: Notes function threw at least one error in a minute
      Namespace: AWS/Lambda
      MetricName: Errors
      Dimensions:
        - Name: FunctionName
          Value: !Ref NotesFunction
      Statistic: Sum
      Period: 60
      EvaluationPeriods: 1
      Threshold: 1
      ComparisonOperator: GreaterThanOrEqualToThreshold
      TreatMissingData: notBreaching
      AlarmActions: !If [HasAlarmEmail, [!Ref AlarmTopic], !Ref AWS::NoValue]

Outputs:
  ApiUrl:
    Value: !GetAtt HttpApi.ApiEndpoint
  TableName:
    Value: !Ref NotesTable
  FunctionName:
    Value: !Ref NotesFunction
  LogGroup:
    Value: !Ref NotesLogGroup
EOF
aws cloudformation validate-template --template-body file://template.yaml --query Description --output text
```

**What you should see:** the template's description, beginning `Capstone: a serverless notes API.`

**If it fails:** a YAML error almost always means the paste was cut off or re-indented. Run `tail -5 template.yaml`:
the last line should be `    Value: !Ref NotesLogGroup`. If it isn't, delete the file (`rm template.yaml`) and paste
again. (Alternative: download `template.yaml` from GitHub, then in CloudShell use **Actions → Upload file**, and run
`mv ~/template.yaml ~/labs/capstone/`.)

---

## Part 3: Read the template (5 min)

You should recognize every part. Here's each resource, with the module where you learned it:

| Logical ID | Type | What it does | Module |
|---|---|---|---|
| `AlarmEmail` (parameter) | | Optional email for alarm notifications. `Conditions: HasAlarmEmail` is true only if you give one. | 07 |
| `NotesTable` | `AWS::DynamoDB::Table` | On-demand table with partition key `id` (no sort key: each note is fetched by its ID). | 05 |
| `NotesFunctionRole` | `AWS::IAM::Role` | **Trust policy:** only Lambda may assume it. **Permissions:** `AWSLambdaBasicExecutionRole` (write logs), plus an inline policy allowing **only** GetItem, PutItem, DeleteItem, and Scan, **only** on `NotesTable` (`!GetAtt NotesTable.Arn`). Least privilege. | 02, 06 |
| `NotesLogGroup` | `AWS::Logs::LogGroup` | Where the function's logs go, **kept 7 days** instead of the default "forever". | 07 |
| `NotesFunction` | `AWS::Lambda::Function` | The code (below). `Architectures: [arm64]` runs on Graviton, which is cheaper per millisecond. The table name is passed in as an environment variable with `!Ref NotesTable`. | 04, 06 |
| `HttpApi` | `AWS::ApiGatewayV2::Api` | The HTTP API. `Target: !GetAtt NotesFunction.Arn` sends every request to the function. | 08 |
| `ApiInvokePermission` | `AWS::Lambda::Permission` | Resource-based policy on the function: `apigateway.amazonaws.com` may invoke it, but **only** from this API (`SourceArn`). | 02 |
| `AlarmTopic` | `AWS::SNS::Topic` | Email subscription. Only created if you gave an email (`Condition: HasAlarmEmail`). | 07 |
| `ErrorsAlarm` | `AWS::CloudWatch::Alarm` | Goes to `ALARM` if the function's `Errors` metric is ≥ 1 in any minute. Notifies the topic, if there is one. | 07 |

**The function code** (the `ZipFile: |` block; `|` in YAML means "the following indented lines are one block of text"):
- `table = boto3.resource("dynamodb").Table(os.environ["TABLE_NAME"])` is outside the handler, so it runs once per cold start (Module 06).
- `respond(status, body)` builds the response API Gateway expects: `statusCode`, `headers`, and a `body` that is a **string** of JSON.
- `handler` reads the HTTP **method** and **path** from the event, splits the path into parts, and decides:
  - `GET /notes` → Scan the table (fine for a small demo; Module 05 explains why not at scale)
  - `POST /notes` → read the JSON body, create an item with a random 8-character `id`, return `201`
  - `GET /notes/{id}` → GetItem; `404` if missing
  - `DELETE /notes/{id}` → DeleteItem, return `204`
  - `/boom` → **deliberately crashes**, so you can watch the alarm fire
  - anything else → `404` or `405`
- Every request prints one line of JSON (`level`, `method`, `path`). **Structured logs** like this can be searched by field (Part 6).

> Putting code inline (`ZipFile`) only works for small functions (up to 4 KB). Real projects keep code in separate
> files and use **AWS SAM** or **CDK** to package and upload it. The rest of the template stays the same.

---

## Part 4: Deploy (2 min)

**Replace the email with yours**, then run:
```bash
MY_EMAIL=you@example.com
aws cloudformation deploy \
  --stack-name notes-api \
  --template-file template.yaml \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides AlarmEmail=$MY_EMAIL
```

**What you should see** (after about 1 minute):
```
Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - notes-api
```

**What just happened:** CloudFormation created eight resources in dependency order: the table and role first, then
the log group and function, then the API and its permission, and the topic and alarm.
`--capabilities CAPABILITY_IAM` is you explicitly acknowledging that this template creates IAM resources. Without it
CloudFormation refuses, as a safety check, because IAM changes can grant powerful access.

**Now confirm the email subscription** (subject *"AWS Notification - Subscription Confirmation"* → **Confirm subscription**).

**If it fails:** `Requires capabilities : [CAPABILITY_IAM]` means you left out that line. For any other failure, look at
the **Events** tab of the `notes-api` stack in the CloudFormation console. The first red `CREATE_FAILED` row says why.
Then delete the stack (Part 8) and deploy again.

**Save the outputs:**
```bash
API=$(aws cloudformation describe-stacks --stack-name notes-api \
  --query "Stacks[0].Outputs[?OutputKey=='ApiUrl'].OutputValue" --output text)
LOG_GROUP=$(aws cloudformation describe-stacks --stack-name notes-api \
  --query "Stacks[0].Outputs[?OutputKey=='LogGroup'].OutputValue" --output text)
TABLE=$(aws cloudformation describe-stacks --stack-name notes-api \
  --query "Stacks[0].Outputs[?OutputKey=='TableName'].OutputValue" --output text)
save API; save LOG_GROUP; save TABLE
```
**What you should see:** three `saved:` lines. `API` looks like `https://a1b2c3d4e5.execute-api.us-east-1.amazonaws.com`.

---

## Part 5: Use the API (3 min)

### Step 1: Create two notes
```bash
curl -s -X POST $API/notes -H 'content-type: application/json' -d '{"text": "IAM: deny by default"}'; echo
curl -s -X POST $API/notes -H 'content-type: application/json' -d '{"text": "Always use 2+ AZs"}'; echo
```
**What you should see:** each returns the created note, e.g. `{"id": "3f9a1c2b", "text": "IAM: deny by default", "createdAt": 1790550000}`.

**What `curl` is doing:** `-X POST` sets the HTTP method, `-H` adds a header saying the body is JSON, `-d` is the body,
and `-s` hides the progress bar. `; echo` just adds a newline after the output.

### Step 2: List them
```bash
curl -s $API/notes; echo
```
**What you should see:** a JSON list containing both notes.

### Step 3: Get, then delete, one note
```bash
ID=$(curl -s $API/notes | python3 -c 'import sys, json; print(json.load(sys.stdin)[0]["id"])')
echo "Working with note $ID"
curl -s $API/notes/$ID; echo
curl -s -o /dev/null -w "DELETE returned HTTP %{http_code}\n" -X DELETE $API/notes/$ID
curl -s -w "\nGET after delete returned HTTP %{http_code}\n" $API/notes/$ID
```
**What you should see:**
```
Working with note 3f9a1c2b
{"id": "3f9a1c2b", "text": "IAM: deny by default", "createdAt": 1790550000}
DELETE returned HTTP 204
{"error": "not found"}
GET after delete returned HTTP 404
```
**What's new here:** a short Python one-liner reads the JSON list and prints the first note's `id`. `-w "...%{http_code}"`
makes curl print the HTTP status code (Module 00, 2.4), and `-o /dev/null` throws away the body.

### Step 4: Open it in your browser
Run `echo $API/notes` and open that URL in your browser. You'll see the JSON list. Your API is live on the internet.

### Step 5: Look at the data directly
```bash
aws dynamodb scan --table-name $TABLE --query 'Items[].[id.S, text.S]' --output table
```
**What you should see:** the remaining note, stored in DynamoDB exactly as the API returned it.

---

## Part 6: Operate it: logs, errors, alarms (5 min)

### Step 1: Break it on purpose
```bash
for i in 1 2 3; do curl -s -w "  (HTTP %{http_code})\n" $API/boom; done
```
**What you should see:** three lines of `{"message":"Internal Server Error"}  (HTTP 500)`.

**What just happened:** the function raised an exception. API Gateway turned the failure into an HTTP `500` for the
caller, and Lambda recorded **3 errors** in the `Errors` metric that the alarm watches.

### Step 2: Find the error in the logs
```bash
aws logs tail $LOG_GROUP --since 10m --format short | grep -A 3 ERROR | head -20
```
**What you should see:** `[ERROR] RuntimeError: intentional failure to trigger the alarm`, followed by the
**traceback**: the file and line where it happened.

### Step 3: Watch the alarm fire
```bash
ALARM=$(aws cloudwatch describe-alarms --alarm-name-prefix notes-api --query 'MetricAlarms[0].AlarmName' --output text)
echo "Alarm: $ALARM"
until [ "$(aws cloudwatch describe-alarms --alarm-names "$ALARM" --query 'MetricAlarms[0].StateValue' --output text)" = "ALARM" ]; do
  echo "Alarm is not in ALARM yet, checking again in 20s..."; sleep 20
done; echo "ALARM! Check your email."
```
**What you should see:** a few "checking again" lines (usually 1-3 minutes, as Lambda metrics arrive), then
`ALARM! Check your email.` An email titled *"ALARM: notes-api-ErrorsAlarm-..."* arrives shortly after.

**What just happened:** this is the full production feedback loop. An error in your code became a metric, the metric
crossed a threshold, the alarm changed state, SNS emailed a human, and the logs showed exactly what failed. In real
teams the email would be a page to the on-call engineer's phone. A few minutes after the errors stop, the alarm goes
back to `OK` (and emails you again if you set `OKActions`).

### Step 4: Search logs with CloudWatch Logs Insights
1. Search `CloudWatch` → left menu **Logs** → **Logs Insights**.
2. In **Select log groups**, choose the one starting with `notes-api-NotesLogGroup`.
3. Set the time range (top-right) to **1h**.
4. Replace the query text with this and click **Run query**:
   ```
   filter @type = "REPORT"
   | stats count(*) as invocations, avg(@duration) as avg_ms, max(@duration) as max_ms, max(@maxMemoryUsed / 1000 / 1000) as max_mb
   ```
   **What you should see:** one row: how many times the function ran, its average and slowest run times, and peak memory.
   Lambda's `REPORT` lines (Module 06, Step 6) are automatically split into fields like `@duration`.
5. Now this query, which uses the JSON fields your code prints:
   ```
   filter ispresent(method)
   | stats count(*) as requests by method, path
   | sort requests desc
   ```
   **What you should see:** a table of request counts per method and path, e.g. `GET /notes`, `POST /notes`, `GET /boom`.
   Logs Insights discovered `method` and `path` automatically because each line is JSON. **This is why you log structured JSON.**

### Step 5: See the whole stack in the console
- **CloudFormation → Stacks → notes-api → Resources**: all eight resources, with links to each.
- **Lambda → Functions** → the `notes-api-NotesFunction-...` function: the diagram shows **API Gateway** as its trigger.
- **API Gateway → APIs → notes-api-api → Routes**: the `$default` route pointing at the function.

---

## Part 7: Stretch challenges (optional, after the course)

Each one uses the same loop: **edit `template.yaml` → preview → deploy**. To edit a file in CloudShell, run `nano template.yaml`:
arrow keys to move, type to edit, **Ctrl+O** then **Enter** to save, **Ctrl+X** to exit. Preview with the `--no-execute-changeset`
option from Module 07, Step 5.

1. **Prove least privilege.** Delete the line `dynamodb:DeleteItem, ` from the role's `Action` list, deploy, then call
   `DELETE /notes/<id>`. You'll get a `500`, and the logs will show the `AccessDeniedException`. Put it back.
2. **Add an update route.** Handle `PUT /notes/{id}` in the code with `table.update_item(...)`, and add `dynamodb:UpdateItem` to the role.
3. **Protect your data.** Add `DeletionPolicy: Retain` to `NotesTable` (at the same indentation as `Type:`). Now deleting the stack keeps the table.
4. **Add authentication.** Create a Cognito user pool and add a JWT authorizer to the HTTP API, so only signed-in users can call it.
5. **Go async.** On `POST`, also send the note to an SQS queue with a DLQ (Modules 06-07), processed by a second function.
6. **Rebuild it in another tool** (AWS CDK or Terraform). Comparing the two versions is the best way to learn IaC.

---

## Part 8: Tear down, then sweep the whole account

### Step 1: Delete the stack
```bash
aws cloudformation delete-stack --stack-name notes-api
aws cloudformation wait stack-delete-complete --stack-name notes-api && echo "notes-api deleted"
cd ~/labs
```
**What you should see:** `notes-api deleted`. The table (and its data), function, API, role, logs, topic, and alarm are all gone.

### Step 2: Final check: is anything from the course still running?

Paste this whole block. It lists every kind of resource the course created:
```bash
echo "== EC2 instances (not terminated) =="
aws ec2 describe-instances --filters Name=instance-state-name,Values=pending,running,stopping,stopped \
  --query 'Reservations[].Instances[].[InstanceId,Tags[?Key==`Name`]|[0].Value]' --output text
echo "== Load balancers ==";            aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerName' --output text
echo "== Target groups ==";             aws elbv2 describe-target-groups --query 'TargetGroups[].TargetGroupName' --output text
echo "== Non-default VPCs ==";          aws ec2 describe-vpcs --filters Name=is-default,Values=false --query 'Vpcs[].VpcId' --output text
echo "== NAT gateways ==";              aws ec2 describe-nat-gateways --filter Name=state,Values=pending,available --query 'NatGateways[].NatGatewayId' --output text
echo "== Elastic IPs ==";               aws ec2 describe-addresses --query 'Addresses[].PublicIp' --output text
echo "== S3 buckets ==";                aws s3 ls
echo "== DynamoDB tables ==";           aws dynamodb list-tables --query 'TableNames' --output text
echo "== Lambda functions ==";          aws lambda list-functions --query 'Functions[].FunctionName' --output text
echo "== SQS queues ==";                aws sqs list-queues --query 'QueueUrls' --output text
echo "== CloudFormation stacks ==";     aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE ROLLBACK_COMPLETE UPDATE_ROLLBACK_COMPLETE --query 'StackSummaries[].StackName' --output text
echo "== IAM roles named lab-* ==";     aws iam list-roles --query "Roles[?starts_with(RoleName,'lab-')].RoleName" --output text
echo "== Instance profiles ==";         aws iam list-instance-profiles --query 'InstanceProfiles[].InstanceProfileName' --output text
echo "== CloudWatch alarms ==";         aws cloudwatch describe-alarms --query 'MetricAlarms[].AlarmName' --output text
echo "== Lambda log groups ==";         aws logs describe-log-groups --log-group-name-prefix /aws/lambda/ --query 'logGroups[].logGroupName' --output text
```

**What you should see:** every heading followed by **an empty line or `None`**. Anything else was left behind:
- Something from this course (names start with `lab-` or `notes-api`): go back to that module's cleanup section and run it.
- Something you don't recognize: you (or someone) created it outside the course. Look before deleting.

This only checks `us-east-1`. If you ever created things with the console set to another region, switch region and look
there too. The **Bills** page (Module 07, Step 10) shows charges by region.

---

## You're done

In about three hours you've:
- Secured an AWS account and set a spending alert
- Written IAM policies and roles, and read permission errors like a pro
- Built a network with public and private subnets, route tables, and security groups
- Run highly available web servers behind a load balancer, and watched them survive a failure
- Used S3 properly (private, versioned, presigned, lifecycle-managed) and designed a DynamoDB table around its queries
- Built a queue-driven serverless pipeline with retries, a dead-letter queue, and idempotency
- Deployed, previewed, changed, and deleted infrastructure as code
- Watched a system with logs, metrics, alarms, and an audit trail
- Deployed a complete API from one file

**Next:** test yourself with the [final quiz](../quiz.md), keep the [cheatsheet](../cheatsheet.md) and [glossary](../glossary.md)
handy, and see the end of the quiz for where to go from here.
