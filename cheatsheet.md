# AWS Cheatsheet: "I need X → use Y"

## Compute
| I need... | Use |
|---|---|
| A virtual machine | **EC2** |
| To run code on events, without servers, runs < 15 min | **Lambda** |
| To run containers without managing servers | **ECS on Fargate** |
| Kubernetes | **EKS** |
| A private Docker registry | **ECR** |
| Batch jobs at scale | **AWS Batch** (on Spot) |
| Keep N instances healthy across AZs, scale on load | **Auto Scaling Group** |
| HTTP load balancing / path routing | **ALB** · TCP/UDP / static IP → **NLB** |

## Storage & data
| I need... | Use |
|---|---|
| Store files/objects, host static sites, data lake | **S3** |
| A disk for an EC2 instance | **EBS** (`gp3`) |
| A shared filesystem across instances | **EFS** |
| Archive for years | **S3 Glacier Deep Archive** |
| Relational DB (Postgres/MySQL...) | **RDS**, or **Aurora** for more scale/availability |
| Key-value/NoSQL at any scale, serverless | **DynamoDB** |
| An in-memory cache | **ElastiCache** (Valkey/Redis OSS) |
| A data warehouse | **Redshift** |
| SQL queries over files in S3 | **Athena** (+ **Glue** Data Catalog) |
| Search / log analytics | **OpenSearch** |
| Many Lambdas connecting to RDS | **RDS Proxy** |

## Networking & delivery
| I need... | Use |
|---|---|
| A private network | **VPC** (subnets per AZ, route tables) |
| Internet access for a public subnet | **Internet Gateway** |
| Outbound-only internet for private subnets | **NAT Gateway** (costs $) |
| Private access to S3/DynamoDB | **Gateway VPC Endpoint** (free) |
| Instance/ENI firewall | **Security Group** (stateful) · subnet firewall → **NACL** (stateless) |
| DNS / domains / health-checked failover | **Route 53** |
| CDN / global caching / HTTPS in front of S3 | **CloudFront** |
| Connect many VPCs | **Transit Gateway** · two VPCs → **VPC Peering** |
| Connect to my datacenter | **Site-to-Site VPN** / **Direct Connect** |

## Integration
| I need... | Use |
|---|---|
| A work queue, buffering, retries | **SQS** (+ DLQ) |
| Fan-out one message to many subscribers | **SNS** |
| Content-based event routing, SaaS events | **EventBridge** |
| Cron / scheduled tasks | **EventBridge Scheduler** |
| Ordered, replayable high-volume streams | **Kinesis Data Streams** / **MSK** (Kafka) |
| Multi-step workflows with retries/compensation | **Step Functions** |
| An HTTP/WebSocket API in front of Lambda | **API Gateway** |
| User sign-up/login, JWTs | **Cognito** |
| Send email | **SES** |

## Security & identity
| I need... | Use |
|---|---|
| Permissions | **IAM** (roles > users; least privilege) |
| Human SSO across accounts | **IAM Identity Center** |
| Manage many accounts, guardrails | **Organizations** + **SCPs** (or **Control Tower**) |
| Encryption keys | **KMS** |
| Store/rotate secrets | **Secrets Manager** · simple config → **SSM Parameter Store** |
| TLS certificates | **ACM** |
| Web firewall / rate limiting | **WAF** · DDoS → **Shield** |
| Threat detection | **GuardDuty** · findings rollup → **Security Hub** |
| Vulnerability scanning | **Inspector** · PII in S3 → **Macie** |

## Operations
| I need... | Use |
|---|---|
| Metrics, logs, alarms, dashboards | **CloudWatch** |
| Distributed tracing | **X-Ray** / CloudWatch Application Signals |
| Audit "who did what" | **CloudTrail** |
| Config history & compliance rules | **AWS Config** |
| Shell into instances without SSH | **SSM Session Manager** |
| Infrastructure as code | **CloudFormation** / **CDK** / **SAM** / **Terraform** |
| Budget alerts / cost breakdown | **Budgets** / **Cost Explorer** |

## Numbers worth memorizing
| Fact | Value |
|---|---|
| Lambda max timeout / memory | 15 min / 10 GB |
| API Gateway integration timeout | 29 s (default) |
| DynamoDB max item size | 400 KB |
| S3 max object size | 5 TB (single PUT ≤ 5 GB → use multipart) |
| S3 durability | 11 nines |
| SQS default visibility timeout | 30 s |
| Spot interruption notice | 2 min |
| RDS automated backup retention | up to 35 days |
| CloudTrail event history (free) | 90 days |
| IAM policy version string | `"2012-10-17"` |

## CLI one-liners
```bash
aws sts get-caller-identity                                   # who am I?
aws configure list                                            # which creds/region are active?
aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,State.Name,PublicIpAddress]' --output table
aws logs tail /aws/lambda/$FUNCTION_NAME --follow                        # live Lambda logs
aws cloudformation describe-stack-events --stack-name $STACK_NAME --max-items 10   # why did my deploy fail?
aws iam simulate-principal-policy --policy-source-arn $ROLE_ARN --action-names s3:GetObject --resource-arns $RESOURCE_ARN
```
