# 01 · IAM: Identity & Access Management (30 min)

IAM is the most important service to understand. Every other service goes through it, and most
AWS security incidents come down to IAM mistakes.

## Mental model

Every API call is evaluated as:

> **Can this PRINCIPAL perform this ACTION on this RESOURCE under these CONDITIONS?**

and the answer is **deny by default**. Something has to explicitly allow it.

## The pieces

| Thing | What it is | Example |
|---|---|---|
| **Principal** | Who's making the call | an IAM user, a role session, an AWS service |
| **IAM User** | Long-lived identity with a password and/or access keys | `alice`, or a CI bot (avoid) |
| **IAM Group** | A set of users that share policies | `developers` |
| **IAM Role** | An identity with **no permanent credentials**. You *assume* it and get temporary creds from STS | `ec2-app-role`, `github-deploy-role` |
| **Policy** | A JSON document listing allows/denies | `AmazonS3ReadOnlyAccess` |

**Roles are the key idea.** Your code on EC2/Lambda/ECS doesn't store access keys. It runs
*as a role*, and the SDK fetches rotating temporary credentials automatically. Humans should log
in with SSO and assume roles too. Long-lived access keys are the thing attackers steal from
GitHub repos.

## Anatomy of a policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadReports",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::acme-reports",
        "arn:aws:s3:::acme-reports/*"
      ],
      "Condition": { "IpAddress": { "aws:SourceIp": "203.0.113.0/24" } }
    }
  ]
}
```

- `Version` is always `"2012-10-17"`. It's the policy language version, not a date you change.
- `Action` is `service:ApiName`. Wildcards work (`s3:Get*`).
- `Resource`: note **bucket** (`acme-reports`) vs **objects** (`acme-reports/*`). `ListBucket`
  applies to the bucket and `GetObject` applies to objects. Getting this wrong is the classic S3
  AccessDenied.

## Two kinds of policy you must not confuse

1. **Identity-based policies** attach to a user/group/role and say *what this identity can do*.
2. **Resource-based policies** attach to a resource (S3 bucket policy, SQS queue policy, KMS key
   policy, Lambda permission) and say *who can access this resource*. They have a
   `Principal` field.

And a special resource-based policy every role has:

3. **Trust policy**: says *who is allowed to assume this role*.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "lambda.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```
So a Lambda execution role has a **trust policy** ("Lambda can wear me") plus **permission
policies** ("whoever wears me can write to DynamoDB table X").

## How a request is evaluated (simplified)

```
1. Is there an explicit DENY anywhere?              → DENIED (deny always wins)
2. Does an SCP (Organizations) / permission boundary
   / session policy fail to allow it?                → DENIED
3. Does an identity policy OR a resource policy
   allow it? (same account)                          → ALLOWED
4. Otherwise                                         → DENIED (implicit deny)
```

**Cross-account** access needs *both* sides to agree: the identity policy in account A must
allow it **and** the resource policy (or role trust policy) in account B must allow it.

## Best practices (the ones that matter)

- **Least privilege**: start narrow and widen when you hit AccessDenied. Use **IAM Access
  Analyzer** to generate policies from actual CloudTrail activity.
- **No access keys for workloads.** Use roles: EC2 instance profiles, Lambda execution roles,
  ECS task roles, and for CI/CD, **OIDC federation** (e.g. GitHub Actions assumes a role, no
  stored secret).
- **MFA** for humans, **root locked away**.
- **Managed vs inline policies**: AWS-managed ones (`ReadOnlyAccess`) are convenient but broad.
  Customer-managed policies are reusable and versioned. Inline policies are bound to one identity.

## Lab: create a role, assume it, hit a wall (12 min)

We'll make a read-only role for one bucket, assume it, and confirm writes get denied.

```bash
export AWS_REGION=us-east-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET=iam-lab-$ACCOUNT_ID                       # bucket names are globally unique

aws s3 mb s3://$BUCKET
echo "hello from s3" > hello.txt && aws s3 cp hello.txt s3://$BUCKET/

# 1) Trust policy: anyone in MY account (who also has permission) may assume this role
cat > trust.json <<EOF
{ "Version": "2012-10-17",
  "Statement": [{ "Effect": "Allow",
                  "Principal": { "AWS": "arn:aws:iam::$ACCOUNT_ID:root" },
                  "Action": "sts:AssumeRole" }] }
EOF
aws iam create-role --role-name s3-reader --assume-role-policy-document file://trust.json

# 2) Permission policy: read-only on ONE bucket
cat > read.json <<EOF
{ "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": "s3:ListBucket", "Resource": "arn:aws:s3:::$BUCKET" },
    { "Effect": "Allow", "Action": "s3:GetObject",  "Resource": "arn:aws:s3:::$BUCKET/*" } ] }
EOF
aws iam put-role-policy --role-name s3-reader --policy-name read-one-bucket --policy-document file://read.json

sleep 10   # IAM is eventually consistent; new roles take a few seconds to propagate

# 3) Assume it: STS hands back temporary credentials
read AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN < <(
  aws sts assume-role --role-arn arn:aws:iam::$ACCOUNT_ID:role/s3-reader --role-session-name lab \
    --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' --output text)
export AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN

aws sts get-caller-identity           # Arn is now ...:assumed-role/s3-reader/lab
aws s3 ls s3://$BUCKET                # ✅ works
aws s3 cp s3://$BUCKET/hello.txt -    # ✅ works
aws s3 cp hello.txt s3://$BUCKET/x    # ❌ AccessDenied (no s3:PutObject: implicit deny)
aws s3 ls                             # ❌ AccessDenied (no s3:ListAllMyBuckets)

# 4) Drop back to your own identity
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
```

**What you just saw:** a principal with no permanent credentials, temporary keys from STS, and
deny by default. This is exactly what happens when Lambda runs your code, just automated.

### Cleanup
```bash
aws iam delete-role-policy --role-name s3-reader --policy-name read-one-bucket
aws iam delete-role --role-name s3-reader
aws s3 rb s3://$BUCKET --force
rm -f trust.json read.json hello.txt
```

## Check yourself

1. A user's identity policy allows `s3:*` on `*`. The bucket policy has an explicit `Deny` for
   that user on `s3:DeleteObject`. Can they delete objects?
2. Your Lambda gets `AccessDenied` calling `dynamodb:PutItem`. Where do you fix it, the trust
   policy or the permission policy?
3. Why is an IAM role better than putting access keys in your EC2 app's config file?
4. A policy allows `s3:GetObject` on `arn:aws:s3:::photos`. Why does `GetObject` still fail?

<details><summary>Answers</summary>

1. No. Explicit deny always wins.
2. The permission policy on the execution role. The trust policy only controls *who can assume* the role, and Lambda clearly already can, since the function is running.
3. Role credentials are temporary and auto-rotated, never written to disk or git, and can be revoked centrally. Keys in a file are long-lived and easy to leak.
4. Objects are `arn:aws:s3:::photos/*`. The ARN given is the bucket itself.
</details>

**Next → [02 Networking](02-networking-vpc.md)**
