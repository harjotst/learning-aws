# 00 · Setup & the Big Picture (15 min)

## Mental model

AWS is **a huge set of APIs** that create and manage infrastructure. The web console, the
CLI, the SDKs, CloudFormation, and Terraform are all just different clients for the same APIs.
Every action (launching a server, reading a file from S3) is an HTTPS API call that is:

1. **Authenticated**: who are you? (signed with your credentials)
2. **Authorized**: are you allowed? (IAM policies, covered in module 01)
3. **Scoped to a region**: *where* does it happen? (mostly)
4. **Logged**: CloudTrail records it
5. **Billed**: if it creates something that costs money

Once that clicks, AWS stops feeling like 200 separate products.

## Global infrastructure

```
AWS
 └── Region (e.g. us-east-1, N. Virginia)      ← fully independent; you pick it
      ├── Availability Zone us-east-1a          ← 1+ physically separate datacenters
      ├── Availability Zone us-east-1b             with independent power/network,
      └── Availability Zone us-east-1c             low-latency links between them
 └── Edge locations (hundreds)                  ← CloudFront CDN, Route 53 DNS
```

- **Region**: a geographic area. Resources in one region don't automatically exist in another.
  You pick a region for **latency** (close to users), **compliance** (data residency),
  **price** (varies by region), and **feature availability** (new services launch in us-east-1 first).
- **Availability Zone (AZ)**: an isolated failure domain within a region. **High availability
  on AWS = spreading across ≥2 AZs.** This is the most important architecture rule in AWS.
- **Global services**: a few services aren't regional: **IAM**, **Route 53**, **CloudFront**,
  **Organizations**. (S3 bucket *names* are global, but each bucket lives in one region.)

## Accounts

- An **AWS account** is a hard boundary for billing, security, and resource limits.
- The **root user** (the email you signed up with) can do *anything*, including closing the
  account. Turn on MFA, then never use it for daily work.
- Real companies use **AWS Organizations** to run many accounts (dev / staging / prod / security /
  logging), with **IAM Identity Center** (formerly AWS SSO) for human logins. Multi-account is
  the norm, not an advanced topic.

## Shared responsibility model

| AWS is responsible for "security **of** the cloud" | You are responsible for "security **in** the cloud" |
|---|---|
| Physical datacenters, hardware, hypervisor | Your IAM policies and who has access |
| The managed service software (e.g. the RDS engine patches) | Your data, encryption choices, S3 bucket policies |
| Global network | Security groups, OS patching on EC2 |

The more managed the service, the more AWS takes on. With EC2 you patch the OS; with Lambda
you don't.

## Lab: set up safely (10 min)

### 1. Account hygiene (console)
1. Sign in as root → **enable MFA** on root (IAM → Security recommendations).
2. **Billing → Budgets → Create budget → "Zero spend budget"** or a monthly budget of $5 with
   an email alert. Do this *first*. It's the single best protection against a surprise bill.
3. Create an admin identity for yourself. The best option is **IAM Identity Center** (gives you
   short-lived credentials). The quick option for a learning account is an IAM user with the
   `AdministratorAccess` policy, MFA on, and access keys for the CLI.

> Note: AWS changed the Free Tier in July 2025. New accounts get credits (up to $200) and a
> "free plan" for 6 months instead of the old 12-month per-service allowances. Either way, the
> budget alert is what protects you.

### 2. Install and configure the CLI (v2)

```bash
# macOS
brew install awscli
# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip && unzip awscliv2.zip && sudo ./aws/install

aws --version                  # aws-cli/2.x

# Option A: IAM Identity Center (recommended)
aws configure sso              # follow the prompts; creates a named profile
aws sso login --profile <name>

# Option B: access keys (learning account only)
aws configure                  # paste key id + secret, region: us-east-1, output: json
```

### 3. Prove it works

```bash
aws sts get-caller-identity
```
```json
{ "UserId": "AIDA...", "Account": "123456789012", "Arn": "arn:aws:iam::123456789012:user/you" }
```

This is the "whoami" of AWS. When anything permission-related breaks, run it first.
**ARNs** (Amazon Resource Names) are how AWS names everything:
`arn:aws:<service>:<region>:<account>:<resource>`, e.g. `arn:aws:s3:::my-bucket` (S3 is global
so region/account are empty) or `arn:aws:lambda:us-east-1:123456789012:function:hello`.

### 4. Useful CLI habits

```bash
export AWS_REGION=us-east-1          # default region for this shell
export AWS_PROFILE=myprofile         # which credentials to use
aws ec2 describe-regions --query 'Regions[].RegionName' --output table   # --query is JMESPath
aws s3 ls help                       # every command has help
```

`--query` (filter output) and `--output table|text|json` will save you hours.

## Check yourself

1. Your app runs on one EC2 instance in us-east-1a. Is it highly available?
2. You created an IAM user in eu-west-1. Can you use it in us-east-1?
3. You deploy to RDS. Who patches the database engine? Who decides whether it's publicly reachable?

<details><summary>Answers</summary>

1. No. One AZ failure (or one instance failure) takes it down. You need ≥2 instances across ≥2 AZs behind a load balancer.
2. Yes. IAM is global; there's no region to create it in.
3. AWS applies engine patches (you choose the maintenance window). *You* decide network exposure (subnet, security group, `PubliclyAccessible`).
</details>

**Next → [01 IAM](01-iam.md)**
