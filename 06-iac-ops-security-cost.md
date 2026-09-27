# 06 · IaC, Observability, Security & Cost (15 min)

Four topics that separate a demo from production, covered quickly.

---

## Infrastructure as Code (IaC)

**Mental model:** you describe the infrastructure you want in files and a tool makes reality match
them. Those files go in git and get reviewed, and you can recreate everything identically in
another account or region. Clicking around the console is for learning and debugging, not for production.

| Tool | What it is | Notes |
|---|---|---|
| **CloudFormation** | AWS-native. YAML/JSON templates → **stacks** | Tracks state for you, rolls back on failure, handles deletion order. **Change sets** preview updates. |
| **AWS SAM** | CloudFormation extension for serverless | Short syntax for Lambda/API/DynamoDB, plus `sam local` for testing |
| **AWS CDK** | Write TypeScript/Python/etc. that *generates* CloudFormation | Loops, types, and reusable constructs. Popular with app developers. |
| **Terraform / OpenTofu** | Multi-cloud, HCL language, its own state file | The industry-wide standard, especially for platform teams |

A CloudFormation template has three main parts:
```yaml
Parameters:   # inputs (stage name, instance size...)
Resources:    # the actual things (the only required section)
  MyQueue:
    Type: AWS::SQS::Queue
    Properties: { VisibilityTimeout: 60 }
Outputs:      # values to print or export (URLs, ARNs)
  QueueUrl: { Value: !Ref MyQueue }
```
Intrinsic functions: `!Ref` (a resource's ID), `!GetAtt Res.Arn` (an attribute), `!Sub "arn:...${AWS::AccountId}"` (string interpolation).
The capstone (module 07) deploys a real template.

**Drift**: someone changes a resource by hand in the console, so reality no longer matches the
code. CloudFormation can detect drift. The fix is discipline: all changes go through code.

---

## Observability

| Service | Answers the question |
|---|---|
| **CloudWatch Metrics** | "How is it performing?" CPU, request counts, Lambda errors/duration, queue depth. Custom metrics too. |
| **CloudWatch Logs** | "What did the app print?" Lambda/ECS logs land here automatically. Set **retention** (the default is *never expire*, which costs money over time). |
| **CloudWatch Logs Insights** | SQL-ish queries over logs: `fields @timestamp, @message \| filter @message like /ERROR/ \| sort @timestamp desc` |
| **CloudWatch Alarms** | "Tell me when X crosses a threshold" → SNS (email/pager) or Auto Scaling actions |
| **X-Ray / CloudWatch Application Signals** | "Where did the time go across services?" Distributed tracing (OpenTelemetry-compatible) |
| **CloudTrail** | "**Who** did **what** to **which resource**, **when**?" Every API call in the account. Your audit log and first stop in incident response. |
| **AWS Config** | "What did this resource's config look like last Tuesday? Is anything non-compliant?" |

Alarms worth having on almost any system: 5xx rate on the ALB/API, Lambda `Errors` and
`Throttles`, **DLQ depth > 0**, RDS CPU and free storage, and billing.

---

## Security services (know what each is for)

| Service | Purpose |
|---|---|
| **KMS** | Manages encryption keys. Most services encrypt at rest with KMS keys, and **key policies** control who can decrypt. |
| **Secrets Manager** | Stores DB passwords and API keys, with **automatic rotation**. Apps fetch secrets at runtime and nothing sits in env files. |
| **SSM Parameter Store** | Config values and simple secrets (SecureString). Cheaper, no built-in rotation. |
| **ACM** | Free public TLS certificates for ALB/CloudFront/API Gateway, renewed automatically. |
| **WAF** | Web application firewall in front of CloudFront/ALB/API Gateway (SQL injection, bots, rate limits). |
| **Shield** | DDoS protection (Standard is free and always on). |
| **GuardDuty** | Threat detection from CloudTrail, DNS, and flow logs ("these credentials are being used from a TOR exit node"). Turn it on. |
| **Security Hub** | Rolls up findings and best-practice checks across the account. |
| **Inspector** | Scans EC2, container images, and Lambda for CVEs. |
| **Macie** | Finds sensitive data (PII) sitting in S3. |
| **IAM Access Analyzer** | Finds resources shared outside your account, and generates least-privilege policies. |

**The baseline for a new account:** root MFA, no root access keys, SSO for humans, CloudTrail
on (org-wide), GuardDuty on, S3 Block Public Access on at the account level, default EBS
encryption on, and budgets set.

---

## Cost

**Mental model:** you pay for **what's provisioned and running** (instances, NAT gateways, load
balancers, provisioned capacity) plus **what you use** (requests, GB stored, GB transferred).
Serverless moves almost everything into the second bucket.

Where surprise bills come from:
1. **NAT Gateway**: hourly charge plus per-GB processing. Use VPC endpoints for S3/DynamoDB traffic.
2. **Forgotten resources**: an idle RDS instance, an unattached EBS volume, old snapshots, unused Elastic IPs.
3. **Public IPv4 addresses**: every public IPv4 costs about $3.60/month (since 2024).
4. **Data transfer**: *into* AWS is free; *out* to the internet and *between AZs/regions* is not.
5. **CloudWatch Logs** kept forever, and verbose debug logging.
6. **Leaked access keys** used for crypto mining. (This is why budgets alert and why workloads should use roles, not keys.)

Tools: **Budgets** (alerts), **Cost Explorer** (where the money goes, grouped by service or
tag), **cost allocation tags** (tag everything with `project`/`env`/`owner`), **Compute
Optimizer** (right-sizing), and **Savings Plans** for steady compute.

---

## The Well-Architected Framework (6 pillars)

AWS's checklist for reviewing designs. Knowing the pillar names lets you structure any design discussion:

1. **Operational Excellence**: IaC, observability, small reversible changes
2. **Security**: least privilege, encryption, traceability, defense in depth
3. **Reliability**: multi-AZ, auto-recovery, backups, tested restores, quotas
4. **Performance Efficiency**: right services, right sizes, serverless, caching
5. **Cost Optimization**: pay for use, right-size, Spot/Savings Plans, turn things off
6. **Sustainability**: efficient resource use, Graviton, managed services

---

## Mini-lab: look at your own audit trail (3 min)

You've made dozens of API calls in the previous labs. CloudTrail recorded all of them, free, for 90 days:

```bash
aws cloudtrail lookup-events --max-results 15 \
  --query 'Events[].[EventTime,EventName,Username]' --output table

# Who deleted things? (answer: you, during cleanup)
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteQueue \
  --query 'Events[].[EventTime,Username]' --output table

# Did you set a budget in module 00?
aws budgets describe-budgets --account-id $(aws sts get-caller-identity --query Account --output text) \
  --query 'Budgets[].[BudgetName,BudgetLimit.Amount]' --output table
```

## Check yourself

1. An S3 bucket was made public last night. Which service tells you who did it?
2. Where should your Lambda get its database password from?
3. Your bill jumped $40 this month, and the line item is "NAT Gateway bytes". Likely cause and fix?
4. Why prefer CloudFormation/Terraform over the console even for a small project?

<details><summary>Answers</summary>

1. CloudTrail (`PutBucketPolicy` / `PutPublicAccessBlock` events). AWS Config shows the configuration change, and Security Hub/GuardDuty/Access Analyzer would flag the exposure.
2. Secrets Manager (with rotation), fetched at init and cached. The execution role gets `secretsmanager:GetSecretValue` on that one secret.
3. Private-subnet workloads pulling lots of data (often from S3 or ECR) through NAT. Add gateway VPC endpoints for S3/DynamoDB, and interface endpoints for ECR where volume justifies it.
4. It's reproducible, reviewable, versioned, and deletable as one unit (no forgotten resources), and you can create identical dev and prod environments.
</details>

**Next → [07 Capstone](07-capstone/README.md)**
