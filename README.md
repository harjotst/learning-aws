# AWS in 3 Hours: A Step-by-Step Course

This course assumes **no prior knowledge**. Every term is explained the first time it appears,
and every lab step tells you exactly what to type or click, what you should see, what just
happened, and what to do if it doesn't work.

## How the course works

**One continuous project.** Each module builds on the one before it, so nothing appears out of
nowhere:

```
00 Foundations      what servers, IPs, ports, DNS, JSON and the terminal are
01 Account setup    create the account, secure it, open CloudShell (your terminal)
02 IAM              create a storage bucket + a permission "role" for a web server
03 Networking       build a private network (VPC) for the web server
04 Compute          launch 2 web servers in that network, using that role,
                    behind a load balancer, then break one and watch it survive
05 Storage & DBs    go deeper on the bucket; create a database table
06 Serverless       write a Lambda function that processes orders from a queue
                    and saves them into that table
07 IaC & operations recreate infrastructure from a file; logs, metrics, alarms, audit, cost
08 Capstone         deploy a complete API → function → database app with one command
```

**Every lab runs in AWS CloudShell**, a terminal inside the AWS website that's already logged in.
You don't need to install anything, and it works the same on Windows, Mac, and Linux.

**Every lab step has the same four parts:**
1. **Do this**: the exact command or clicks
2. **What you should see**: sample output, so you know it worked
3. **What just happened**: the explanation
4. **If it fails**: the common errors and fixes

## Schedule (about 3 hours)

| Start | Module | Time |
|---|---|---|
| 0:00 | [00 · Foundations](00-foundations.md) | 20 min |
| 0:20 | [01 · Account Setup](01-account-setup.md) | 20 min |
| 0:40 | [02 · IAM: Identity & Access](02-iam.md) | 25 min |
| 1:05 | [03 · Networking: VPC](03-networking-vpc.md) | 25 min |
| 1:30 | [04 · Compute: EC2 & Load Balancers](04-compute-ec2.md) | 30 min |
| 2:00 | [05 · Storage & Databases: S3 & DynamoDB](05-storage-databases.md) | 20 min |
| 2:20 | [06 · Serverless: Lambda & SQS](06-serverless-lambda-sqs.md) | 20 min |
| 2:40 | [07 · Infrastructure as Code & Operations](07-iac-and-operations.md) | 15 min |
| 2:55 | [08 · Capstone](08-capstone/README.md) | 15 min |
| after | [Final quiz](quiz.md) · [Cheatsheet](cheatsheet.md) · [Glossary](glossary.md) | |

If you're short on time, sections marked **(optional)** can be skipped without breaking later modules.

## Cost

If you run each module's **cleanup** section, the whole course costs **well under $1**, and new
AWS accounts get free credits that cover it. Module 01 sets up a budget alert before you
create anything.
