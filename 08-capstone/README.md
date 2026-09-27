# 07 · Capstone: Deploy a Serverless API with One Command (10+ min)

You'll deploy [`template.yaml`](template.yaml), a production-shaped mini app that uses most of what you've learned:

```
curl ──HTTPS──▶ API Gateway (HTTP API) ──▶ Lambda (Python, arm64) ──▶ DynamoDB
                                              │    runs as an IAM role scoped to ONE table
                                              ├──▶ CloudWatch Logs (7-day retention)
                                              └──▶ CloudWatch Alarm on Errors ──▶ SNS ──▶ your email
```

| Module | Where it appears in the template |
|---|---|
| 01 IAM | `NotesFunctionRole`: trust policy + least-privilege permission policy. `ApiInvokePermission`: a resource-based policy |
| 03 Compute | `NotesFunction`: Lambda, init code outside the handler, Graviton |
| 04 Databases | `NotesTable`: DynamoDB on-demand, partition key `id` |
| 05 Integration | `HttpApi`: API Gateway HTTP API |
| 06 IaC / Ops | The whole file is a CloudFormation stack, plus `NotesLogGroup` and `ErrorsAlarm` |

Read the template top to bottom before deploying. It's about 150 lines, and you should now recognize every resource in it.

## Deploy

```bash
cd 07-capstone
aws cloudformation deploy \
  --stack-name notes-api \
  --template-file template.yaml \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides AlarmEmail=you@example.com     # optional; confirm the email AWS sends

API=$(aws cloudformation describe-stacks --stack-name notes-api \
  --query "Stacks[0].Outputs[?OutputKey=='ApiUrl'].OutputValue" --output text)
echo $API
```

`--capabilities CAPABILITY_IAM` is you acknowledging that the stack creates IAM resources. It's a deliberate speed bump.

While it deploys (~1 min), watch it in the console: **CloudFormation → notes-api → Events**.

## Use it

```bash
ID=$(curl -s -X POST $API/notes -H 'content-type: application/json' -d '{"text":"learn IAM"}' \
     | python3 -c 'import sys,json;print(json.load(sys.stdin)["id"])')
curl -s -X POST $API/notes -H 'content-type: application/json' -d '{"text":"learn VPC"}'
curl -s $API/notes                       # list
curl -s $API/notes/$ID                   # get one
curl -s -X DELETE $API/notes/$ID -w '%{http_code}\n'   # 204
```

## Operate it

```bash
FN=$(aws cloudformation describe-stacks --stack-name notes-api \
  --query "Stacks[0].Outputs[?OutputKey=='FunctionName'].OutputValue" --output text)
LG=$(aws cloudformation describe-stacks --stack-name notes-api \
  --query "Stacks[0].Outputs[?OutputKey=='LogGroup'].OutputValue" --output text)

# 1) Live logs
aws logs tail $LG --follow &

# 2) Break it on purpose → 500 + Lambda error → alarm goes to ALARM within ~1-2 min
for i in 1 2 3; do curl -s $API/boom; echo; done
aws cloudwatch describe-alarms --alarm-name-prefix notes-api \
  --query 'MetricAlarms[].[AlarmName,StateValue]' --output table

# 3) Logs Insights: which paths are being hit?
QID=$(aws logs start-query --log-group-name $LG \
  --start-time $(($(date +%s) - 3600)) --end-time $(date +%s) \
  --query-string 'filter level = "INFO" | stats count(*) by method, path' \
  --query queryId --output text)
sleep 5 && aws logs get-query-results --query-id $QID --query 'results' --output json

# 4) Look at the data directly
aws dynamodb scan --table-name $(aws cloudformation describe-stacks --stack-name notes-api \
  --query "Stacks[0].Outputs[?OutputKey=='TableName'].OutputValue" --output text)

kill %1   # stop tailing
```

The structured `print(json.dumps({...}))` in the function is why step 3 works: CloudWatch
automatically discovers JSON fields like `level`, `method`, and `path`.

## Stretch challenges (do them after the 3 hours)

Edit the template, redeploy with the same `deploy` command, and watch CloudFormation work out the diff.

1. **Prove least privilege**: remove `dynamodb:DeleteItem` from the role, redeploy, and call DELETE. Find the `AccessDeniedException` in the logs.
2. **Add a PUT /notes/{id}** that updates `text` (you'll need `dynamodb:UpdateItem`).
3. **Replace the Scan**: add a sort key or a GSI so "list notes" becomes a Query (module 04).
4. **Go async**: on POST, also send the note to an SQS queue with a DLQ, consumed by a second Lambda (module 05).
5. **Lock it down**: add a JWT authorizer to the HTTP API with a Cognito user pool.
6. **Rewrite it in CDK or Terraform**: the best way to compare IaC tools.
7. **CI/CD**: GitHub Actions deploy using **OIDC** to assume a role (no stored AWS keys).

## Tear down

```bash
aws cloudformation delete-stack --stack-name notes-api
aws cloudformation wait stack-delete-complete --stack-name notes-api
```

That removed every resource in the right order: this is what IaC gets you. (DynamoDB data
goes too. For real data you'd set `DeletionPolicy: Retain` on the table.)

**Final step → [quiz.md](../quiz.md)**
