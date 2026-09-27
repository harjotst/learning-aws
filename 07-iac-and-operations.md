# Module 07 · Infrastructure as Code & Operations

**Time: about 15-20 minutes.**
**Before you start:** open CloudShell and check the region is `us-east-1`. Have your email inbox open in
another tab. You'll receive an alarm email in this module.

**Where you are:** over Modules 02-06 you typed roughly a hundred commands to build things, in a careful
order, and about forty more to delete them, in a *different* careful order. Imagine doing that for a real
company with dev, test, and production copies, and a colleague who clicked something in the console that
nobody wrote down. This module shows how professionals avoid all of that, then how they watch over what's
running and what it costs.

**What you'll do:**
1. Describe a queue, a dead-letter queue, an email notification, and an alarm **in one file**, and create them all with **one command**.
2. Change the file, **preview** exactly what will change, then apply it.
3. Trigger the alarm and receive an email.
4. Use CloudWatch metrics, CloudTrail (the audit log), and the billing tools.
5. Delete everything with **one command**.

---

## Part 1: Concepts (6 min read)

### 1.1 Infrastructure as Code (IaC)

**Infrastructure as Code** means writing down the infrastructure you want in text files, and letting a tool
create, change, and delete the real resources to match. The benefits:
- **Repeatable:** create identical copies for dev, test, and production, or in another region.
- **Reviewable:** the files live in git, so changes are reviewed like code and you have a history of who changed what.
- **Safe changes:** the tool shows you what will change *before* it changes it.
- **Clean deletion:** delete the whole set in the right order, with nothing forgotten.

The main tools:

| Tool | What it is |
|---|---|
| **CloudFormation** | AWS's built-in IaC service. You write **templates** in YAML or JSON. **We use it here.** |
| **AWS SAM** | An extension of CloudFormation with shortcuts for serverless apps (Lambda, API Gateway). |
| **AWS CDK** | Write TypeScript, Python, or another language; it *generates* CloudFormation templates. |
| **Terraform / OpenTofu** | A very popular multi-cloud tool with its own language (HCL). Same ideas, different syntax. |

### 1.2 CloudFormation vocabulary

- **Template**: your YAML file. Its sections:
  - `Parameters`: inputs you pass in when deploying (like function arguments).
  - `Resources`: the things to create. **The only required section.** Each has a **logical ID** (a name
    you choose, used inside the template), a `Type` (e.g. `AWS::SQS::Queue`), and `Properties`.
  - `Outputs`: values to print after deployment, such as URLs.
- **Stack**: one deployed copy of a template. CloudFormation tracks every resource it created for the stack.
  Delete the stack and they're all deleted.
- **Intrinsic functions** connect resources to each other:
  - `!Ref X`: the main identifier of resource `X` (for a queue, its URL; for an SNS topic, its ARN), or the value of parameter `X`.
  - `!GetAtt X.Arn`: a specific attribute of resource `X`, such as its ARN.
  - `!Sub "text ${Something}"`: build a string with values substituted in.
- **Change set**: a preview of what an update will do: which resources get **Added**, **Modified**, or
  **Removed**, and whether a modification requires **Replacement** (deleting and recreating the resource,
  which matters for things holding data).
- **Rollback**: if anything fails during create/update, CloudFormation automatically undoes the changes.
- **Drift**: when someone changes a resource *outside* CloudFormation (e.g. in the console), reality no longer
  matches the template. CloudFormation can detect this. The rule in teams is: **all changes go through the template.**

CloudFormation figures out the **order** itself. If resource B refers to resource A (with `!Ref` or `!GetAtt`),
A is created first and deleted last.

### 1.3 Operations: knowing what your system is doing

| Service | Question it answers |
|---|---|
| **CloudWatch Metrics** | "How is it performing?" Numbers over time: Lambda errors, queue length, CPU. Every AWS service publishes metrics automatically. |
| **CloudWatch Alarms** | "Tell me when something's wrong." An alarm watches one metric and changes state (`OK` → `ALARM`) when it crosses a threshold, then triggers an action such as an email via SNS. |
| **CloudWatch Logs** | "What did my code print?" (You used it in Module 06.) **Logs Insights** lets you search and summarize logs with a query language (Module 08). |
| **CloudTrail** | "**Who** did **what**, **when**?" A record of every API call in your account: every command you've typed in this course is in it. It's the first place to look after a security incident. |
| **AWS Config** | "What did this resource's settings look like last week, and does it follow our rules?" |

Metric vocabulary: a metric lives in a **namespace** (e.g. `AWS/SQS`), has a **name** (e.g.
`ApproximateNumberOfMessagesVisible`), and is identified by **dimensions** (e.g. `QueueName=lab-orders-dlq`).
When you read it, you choose a **statistic** (Sum, Average, Maximum, ...) over a **period** (e.g. 60 seconds).

### 1.4 Security services to know by name

| Service | What it does |
|---|---|
| **KMS** | Creates and controls encryption keys. Most services encrypt data with KMS keys. |
| **Secrets Manager** | Stores passwords and API keys; programs fetch them at runtime; it can rotate them automatically. |
| **ACM** | Free TLS certificates (for `https://`) for load balancers, CloudFront, and API Gateway. |
| **WAF** | Web firewall in front of your site: blocks common attacks and abusive traffic. |
| **GuardDuty** | Watches CloudTrail, DNS, and network logs for signs of attack ("these credentials are being used from an unusual country"). |
| **Security Hub** | Collects security findings and best-practice checks in one place. |

**A good baseline for any account:** MFA on root and all humans, no long-lived access keys, CloudTrail on,
GuardDuty on, S3 Block Public Access on, budgets set.

### 1.5 Cost: where AWS bills come from

You pay for (a) things that **exist and run**: servers, databases, load balancers, NAT Gateways, public IP
addresses; and (b) **usage**: requests, GB stored, GB sent out to the internet. The usual surprise bills are a
**NAT Gateway** left running, **forgotten servers or databases**, **data transfer out**, **logs kept forever**,
and **leaked access keys** used by attackers to mine cryptocurrency. Tools: **Budgets** (alerts), **Cost
Explorer** (charts of where money goes), **tags** (label resources by project to split the bill).

### 1.6 The Well-Architected Framework

AWS's checklist for judging any design, with **six pillars**: **Operational Excellence** (IaC, monitoring),
**Security** (least privilege, encryption), **Reliability** (multiple AZs, backups), **Performance
Efficiency** (right tools and sizes), **Cost Optimization** (pay only for what you need), and
**Sustainability** (use resources efficiently). Every module in this course touched at least one.

---

## Part 2: Lab (12 min)

### Step 1: Write the template

**Do this** (paste the whole block including `EOF`):
```bash
cat > queue-stack.yaml <<'EOF'
AWSTemplateFormatVersion: "2010-09-09"
Description: Module 07 - an orders queue with a dead-letter queue, an alarm, and email alerts

Parameters:
  AlarmEmail:
    Type: String
    Description: Email address that receives alarm notifications
  VisibilityTimeout:
    Type: Number
    Default: 30
    Description: Seconds a received message stays hidden from other consumers

Resources:
  DeadLetterQueue:
    Type: AWS::SQS::Queue
    Properties:
      MessageRetentionPeriod: 1209600

  OrdersQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: !Ref VisibilityTimeout
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt DeadLetterQueue.Arn
        maxReceiveCount: 3

  AlarmTopic:
    Type: AWS::SNS::Topic
    Properties:
      Subscription:
        - Protocol: email
          Endpoint: !Ref AlarmEmail

  DeadLetterQueueAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub "${AWS::StackName}-dlq-not-empty"
      AlarmDescription: Messages are waiting in the dead-letter queue
      Namespace: AWS/SQS
      MetricName: ApproximateNumberOfMessagesVisible
      Dimensions:
        - Name: QueueName
          Value: !GetAtt DeadLetterQueue.QueueName
      Statistic: Maximum
      Period: 60
      EvaluationPeriods: 1
      Threshold: 1
      ComparisonOperator: GreaterThanOrEqualToThreshold
      TreatMissingData: notBreaching
      AlarmActions:
        - !Ref AlarmTopic

Outputs:
  OrdersQueueUrl:
    Value: !Ref OrdersQueue
  DeadLetterQueueUrl:
    Value: !Ref DeadLetterQueue
  AlarmName:
    Value: !Ref DeadLetterQueueAlarm
EOF
```

**Read the template before deploying.** Compare it to Module 06, Step 7, where you built the same queues with commands:
- `Parameters`: `AlarmEmail` has no default, so you must supply it. `VisibilityTimeout` defaults to 30.
- `DeadLetterQueue`: an SQS queue keeping messages 14 days. **No name is given**, so CloudFormation generates a
  unique one like `lab-queue-stack-DeadLetterQueue-AbC123`. That lets you deploy the same template many times.
- `OrdersQueue`: `VisibilityTimeout: !Ref VisibilityTimeout` uses the parameter. `deadLetterTargetArn: !GetAtt DeadLetterQueue.Arn`
  refers to the other queue's ARN. Because of this reference, CloudFormation knows to **create the DLQ first**.
  Compare this to Module 06, where you had to fetch the ARN yourself and escape JSON inside a string.
- `AlarmTopic`: an **SNS topic** (publish/subscribe, Module 06, 1.5) with an **email subscription**.
- `DeadLetterQueueAlarm`: "look at the **Maximum** of the DLQ's `ApproximateNumberOfMessagesVisible` over **60
  seconds**; if it's **≥ 1** for **1** period, go to `ALARM` and notify the topic." `TreatMissingData: notBreaching`
  means "no data counts as fine". `${AWS::StackName}` is a built-in value: the stack's name.
- `Outputs`: print the two queue URLs and the alarm's name.

### Step 2: Validate it

**Do this:**
```bash
aws cloudformation validate-template --template-body file://queue-stack.yaml --query 'Parameters[].ParameterKey'
```
**What you should see:** `["AlarmEmail", "VisibilityTimeout"]`.

**If it fails:** a `ValidationError` with a line number usually means an **indentation** mistake (YAML is
indentation-sensitive, Module 00, 3.2). Re-paste Step 1 carefully.

### Step 3: Deploy it

**Replace the email address with yours**, then run:
```bash
MY_EMAIL=you@example.com
aws cloudformation deploy \
  --stack-name lab-queue-stack \
  --template-file queue-stack.yaml \
  --parameter-overrides AlarmEmail=$MY_EMAIL
```

**What you should see** (after about 30-60 seconds):
```
Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - lab-queue-stack
```

**What just happened:** `deploy` uploaded the template, created a change set, and executed it. CloudFormation
created the four resources in dependency order.

**Now check your email.** You'll have a message from **AWS Notifications** titled *"AWS Notification - Subscription
Confirmation"*. **Click "Confirm subscription"** in it. SNS never emails someone who hasn't agreed to receive it.
(Check spam if it's not there.)

**If it fails:** `Parameters: [AlarmEmail] must have values` means you left out `--parameter-overrides`. If the
stack shows `ROLLBACK_COMPLETE`, run the delete command from Step 11, fix the problem, and deploy again. A stack
that failed on creation can't be updated.

### Step 4: Look at what CloudFormation built

**Do this:**
```bash
aws cloudformation describe-stacks --stack-name lab-queue-stack \
  --query 'Stacks[0].[StackStatus, Outputs]' --output json

aws cloudformation describe-stack-resources --stack-name lab-queue-stack \
  --query 'StackResources[].[LogicalResourceId,ResourceType,ResourceStatus]' --output table
```

**What you should see:** `CREATE_COMPLETE` and the three outputs; then a table of four resources, each `CREATE_COMPLETE`.

**In the console:** search `CloudFormation` → **Stacks** → `lab-queue-stack`:
- **Events** tab: every step it took, in order. **When a deployment fails, this tab tells you why.**
- **Resources** tab: each logical ID mapped to the real resource, with links.
- **Template** tab: the file you deployed.

Save the queue URLs for later steps:
```bash
QUEUE_URL=$(aws cloudformation describe-stacks --stack-name lab-queue-stack \
  --query "Stacks[0].Outputs[?OutputKey=='OrdersQueueUrl'].OutputValue" --output text)
DLQ_URL=$(aws cloudformation describe-stacks --stack-name lab-queue-stack \
  --query "Stacks[0].Outputs[?OutputKey=='DeadLetterQueueUrl'].OutputValue" --output text)
ALARM_NAME=$(aws cloudformation describe-stacks --stack-name lab-queue-stack \
  --query "Stacks[0].Outputs[?OutputKey=='AlarmName'].OutputValue" --output text)
echo "$QUEUE_URL"; echo "$DLQ_URL"; echo "$ALARM_NAME"
```
(`Outputs[?OutputKey=='OrdersQueueUrl']` is a **filter**: "the output whose key equals `OrdersQueueUrl`".)

### Step 5: Change something, and preview it first

Change the visibility timeout from 30 to 60, but **preview** the change instead of applying it immediately.

**Do this:**
```bash
aws cloudformation deploy \
  --stack-name lab-queue-stack \
  --template-file queue-stack.yaml \
  --parameter-overrides VisibilityTimeout=60 \
  --no-execute-changeset

CHANGE_SET=$(aws cloudformation list-change-sets --stack-name lab-queue-stack \
  --query 'Summaries[0].ChangeSetId' --output text)
aws cloudformation describe-change-set --change-set-name $CHANGE_SET \
  --query 'Changes[].ResourceChange.[Action,LogicalResourceId,Replacement]' --output table
```

**What you should see:**
```
+---------+---------------+--------+
|  Modify |  OrdersQueue  |  False |
+---------+---------------+--------+
```

**What just happened:** `--no-execute-changeset` created the change set but didn't apply it. The preview says
exactly one resource will be **modified**, and `Replacement: False` means it will be updated in place (the queue
and its messages survive). You didn't repeat `AlarmEmail`, so its previous value is reused.

> **Always read the Replacement column.** `True` on a database or bucket means CloudFormation would delete and
> recreate it, **losing its data**. That's the main reason to preview changes.

**Apply it:**
```bash
aws cloudformation execute-change-set --change-set-name $CHANGE_SET
aws cloudformation wait stack-update-complete --stack-name lab-queue-stack && echo "Update complete"
aws sqs get-queue-attributes --queue-url $QUEUE_URL --attribute-names VisibilityTimeout \
  --query Attributes.VisibilityTimeout --output text
```
**What you should see:** `Update complete`, then `60`.

### Step 6 (optional, 3 min): Detect drift, i.e. a change made behind CloudFormation's back

**Do this:**
```bash
# Someone "quickly fixes" the queue by hand:
aws sqs set-queue-attributes --queue-url $QUEUE_URL --attributes VisibilityTimeout=5

# Ask CloudFormation to compare reality with the template:
DRIFT_ID=$(aws cloudformation detect-stack-drift --stack-name lab-queue-stack \
  --query StackDriftDetectionId --output text)
until [ "$(aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id $DRIFT_ID \
  --query DetectionStatus --output text)" != "DETECTION_IN_PROGRESS" ]; do sleep 3; done
aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id $DRIFT_ID \
  --query StackDriftStatus --output text
aws cloudformation describe-stack-resource-drifts --stack-name lab-queue-stack \
  --stack-resource-drift-status-filters MODIFIED \
  --query 'StackResourceDrifts[].PropertyDifferences[].[PropertyPath,ExpectedValue,ActualValue]' --output table
```

**What you should see:** `DRIFTED`, then a table showing `/VisibilityTimeout | 60 | 5`.

**What just happened:** CloudFormation noticed the manual change. On a team, drift is how "works in dev, broken in
prod" happens. Put it back:
```bash
aws sqs set-queue-attributes --queue-url $QUEUE_URL --attributes VisibilityTimeout=60
```

### Step 7: Fire the alarm and get an email

The alarm watches the dead-letter queue. There are two ways to test it:

**A) Instantly, by forcing its state.** This is a standard way to test that notifications actually reach people:
```bash
aws cloudwatch set-alarm-state --alarm-name "$ALARM_NAME" \
  --state-value ALARM --state-reason "Testing the notification path"
```
**What you should see:** nothing in CloudShell, but **within a minute an email** titled *"ALARM: lab-queue-stack-dlq-not-empty ..."*.
(You must have confirmed the subscription in Step 3.) The alarm will return to `OK` on its next evaluation, because the DLQ is empty.

**B) For real, by putting a message in the DLQ:**
```bash
aws sqs send-message --queue-url $DLQ_URL --message-body '{"order_id": "O-6666", "total": -5}' \
  --query MessageId --output text
```
SQS publishes its metrics about once a minute, and they can lag a few minutes. Carry on with Step 8, then check:
```bash
aws cloudwatch describe-alarms --alarm-names "$ALARM_NAME" \
  --query 'MetricAlarms[0].[StateValue,StateReason]' --output text
```
**What you should see**, after 1-5 minutes: `ALARM	Threshold Crossed: 1 datapoint [1.0 ...] was greater than or equal to the threshold (1.0).`,
and a second email.

**In the console:** search `CloudWatch` → **Alarms** → `lab-queue-stack-dlq-not-empty` shows the graph with the red threshold line.

### Step 8: Read metrics from Module 06

CloudWatch keeps metrics for 15 months, even after the resource is deleted. See what your Lambda function did in Module 06:

**Do this:**
```bash
for METRIC in Invocations Errors; do
  echo -n "$METRIC in the last 6 hours: "
  aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name $METRIC \
    --dimensions Name=FunctionName,Value=lab-order-worker \
    --start-time $(date -u -d '-6 hours' +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 3600 --statistics Sum \
    --query 'sum(Datapoints[].Sum)' --output text
done
```

**What you should see:** something like `Invocations in the last 6 hours: 8.0` and `Errors in the last 6 hours: 4.0`
(3 manual test invocations + 2 good/duplicate messages + 3 attempts at the bad one = 8; the permission error + 3 bad attempts = 4).
Your exact numbers depend on how many times you ran each step.

**What just happened:** namespace `AWS/Lambda`, metric `Invocations`/`Errors`, dimension `FunctionName=lab-order-worker`,
statistic `Sum`, in 1-hour periods (`3600` seconds), added together with `sum(...)`. `date -u -d '-6 hours'` prints the time
6 hours ago in the format the CLI expects. **In real life you'd put an alarm on `Errors`**, just like the DLQ alarm.

### Step 9: Ask CloudTrail who did what

**Do this:**
```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteRole \
  --query 'Events[].[EventTime,Username,Resources[0].ResourceName]' --output table

aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=admin \
  --max-results 10 \
  --query 'Events[].[EventTime,EventSource,EventName]' --output table
```

**What you should see:** first, a row for each role you deleted in Modules 04 and 06 (`lab-web-server-role`,
`lab-order-worker-role`), with the time and `admin`; then your 10 most recent API calls, such as
`DescribeStacks` on `cloudformation.amazonaws.com`.

**What just happened:** CloudTrail records every API call. The last 90 days are free to search like this. Events
appear within about 5-15 minutes, so the very latest calls might not show yet. When something changes and nobody
admits to it, this is where you look.

### Step 10: Check your costs

**Do this:**
```bash
aws budgets describe-budgets --account-id $ACCOUNT_ID \
  --query 'Budgets[].[BudgetName,BudgetLimit.Amount,CalculatedSpend.ActualSpend.Amount]' --output table
```
**What you should see:** `monthly-5-dollars | 5.0 | 0.0...` (a few cents at most). Spend data can lag by up to a day.

**In the console:** search `Billing` → **Billing and Cost Management**:
- **Home**: month-to-date cost, and a forecast.
- **Bills**: charges broken down by service and region. This is where you'd find a forgotten resource.
- **Cost Explorer**: charts by service, by day, and by tag. For a new account it can take up to 24 hours to have data.
- **Credits** / **Free Tier**: how much of your free allowance you've used.

### Step 11: Delete everything with one command

**Do this:**
```bash
aws cloudformation delete-stack --stack-name lab-queue-stack
aws cloudformation wait stack-delete-complete --stack-name lab-queue-stack && echo "Stack deleted"
aws cloudformation describe-stacks --stack-name lab-queue-stack 2>&1 | tail -1
```

**What you should see:** `Stack deleted`, then `An error occurred (ValidationError) ... Stack with id lab-queue-stack does not exist`.

**What just happened:** both queues, the topic and its subscription, and the alarm were deleted, in the right order,
with nothing left behind. Compare that to the 20-command cleanup at the end of Module 04.

---

## Checkpoint

1. Name three advantages of IaC over typing commands or clicking in the console.
2. What's the difference between `!Ref DeadLetterQueue` and `!GetAtt DeadLetterQueue.Arn`?
3. A change set says `Replace: True` for your database. What does that mean, and what should you do?
4. What is drift, and how do teams prevent it?
5. You get a surprise $40 charge. Which two places do you look first?
6. Someone deleted a production queue last night. Which service tells you who?

<details><summary>Answers</summary>

1. Any three of: repeatable across environments/regions; reviewable and versioned in git; previews changes before applying; creates and deletes everything in the right order; nothing forgotten.
2. `!Ref` returns the resource's main identifier (for an SQS queue, its URL). `!GetAtt ... .Arn` returns a specific attribute, the ARN.
3. CloudFormation would delete the database and create a new one, losing the data. Don't execute it: change the template so the property doesn't need replacement, or plan a migration (and set `DeletionPolicy: Retain` on data resources).
4. Drift is when real resources differ from the template because of manual changes. Teams make all changes through the template (code review + automated deployment) and run drift detection.
5. **Bills** (charges by service and region) and **Cost Explorer**. Then delete or right-size the resource responsible.
6. CloudTrail (look up the `DeleteQueue` event).
</details>

**Where you are now:** you know how to build, change, observe, audit, and delete infrastructure the professional way.
The capstone puts it all together: an entire application (API, function, database, permissions, logs, alarm) from one template.

**Next: [Module 08 · Capstone](08-capstone/README.md)**
