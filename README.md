# AWS in 3 Hours

A crash course that covers the ~20 AWS services behind most real systems. The aim is for you to
understand *why* each one exists, use it from the CLI, and be able to read an architecture
diagram and know what you're looking at.

## How to use this

Each module has four parts:

1. **Mental model**: the one idea that makes the service make sense.
2. **Key facts**: the numbers and rules that come up in real work (and in interviews).
3. **Lab**: commands to run in your own AWS account. **Every lab ends with a cleanup step.**
4. **Check yourself**: questions, with the answers folded away.

If you don't have an AWS account (or would rather not use one), read the labs anyway. The
commands are annotated so you can follow along.

## Schedule (180 min)

| Time      | Module | What you'll be able to do |
|-----------|--------|---------------------------|
| 0:00–0:15 | [00 Setup & the big picture](00-setup.md) | Explain regions/AZs, set up the CLI safely, set a billing alarm |
| 0:15–0:45 | [01 IAM](01-iam.md) | Write a policy, create a role, explain how a request gets allowed or denied |
| 0:45–1:15 | [02 Networking (VPC)](02-networking-vpc.md) | Design a VPC with public/private subnets, debug "why can't X reach Y" |
| 1:15–1:45 | [03 Compute](03-compute.md) | Launch EC2, pick between EC2, Lambda, ECS/Fargate, and EKS |
| 1:45–2:10 | [04 Storage & Databases](04-storage-databases.md) | Use S3 properly, choose RDS vs DynamoDB, model a DynamoDB table |
| 2:10–2:35 | [05 Serverless & Integration](05-serverless-integration.md) | Decouple with SQS/SNS/EventBridge, front Lambda with API Gateway |
| 2:35–2:50 | [06 IaC, Observability, Security, Cost](06-iac-ops-security-cost.md) | Deploy with CloudFormation, find logs/metrics/audit trails, avoid surprise bills |
| 2:50–3:00 | [07 Capstone](07-capstone/README.md) + [Final quiz](quiz.md) | Deploy a real API → Lambda → DynamoDB stack in one command, then tear it down |

Keep [cheatsheet.md](cheatsheet.md) open the whole time. It's the "I need X → use Y" map.

## Cost safety

Everything here fits in the Free Tier / new-account credits **as long as you run the cleanup
steps**. The two things that quietly cost money are **NAT Gateways** and **running EC2/RDS
instances**. The labs call them out when they come up.
