# Module 02 · IAM: Who Is Allowed to Do What

**Time: about 25 minutes.**
**Before you start:** open CloudShell (Module 01, Part 7) and check you're in `us-east-1`.

**Where you are:** you have an account and a terminal. Before you build anything that runs, you
need to understand the system that approves or rejects **every single action** in AWS. Nearly
every error you hit in AWS that isn't a typo is a permission error, so this module pays for
itself many times over.

**What you'll build (and keep for Module 04):**
1. An **S3 bucket** (a storage container) holding the web page your servers will show.
2. An **IAM role** called `lab-web-server-role`. In Module 04, your web servers will "wear"
   this role so they can read that web page from the bucket **without any password or key**.
3. You'll then wear the role yourself to prove it can read but **can't** write or delete.

---

## Part 1: Concepts (10 min read)

### 1.1 The one question IAM answers

Every request to AWS (from the console, the CLI, or a program) is checked by **IAM (Identity and
Access Management)**. IAM asks:

> **Is this *principal* allowed to do this *action* on this *resource*?**

- **Principal**: *who* is asking (a user, a role, or an AWS service).
- **Action**: *what* they want to do, written `service:Operation`, e.g. `s3:GetObject` (read a file from S3).
- **Resource**: *which thing*, identified by its ARN (see Module 00, 5.3).

And the answer starts at **no**. **Everything is denied unless something explicitly allows it.**
This is called **implicit deny** or **deny by default**.

Two words you'll see:
- **Authentication** means proving who you are (password, MFA code, access key, or temporary credentials).
- **Authorization** means deciding what you're allowed to do (policies).

### 1.2 The four kinds of identity

| Identity | What it is | Has permanent credentials? | Use it for |
|---|---|---|---|
| **Root user** | The account owner (Module 01) | Yes (email + password) | Almost nothing. Locked away. |
| **IAM user** | A named identity, like your `admin` | Yes (password and/or **access keys**) | People, in small setups |
| **IAM group** | A set of IAM users that share permissions | n/a (it's just a container) | "All developers get X" |
| **IAM role** | An identity that nobody "is". Someone or something **temporarily takes it on** | **No** | Programs, AWS services, cross-account access, and people in larger setups |

**Roles are the key idea of this module, so take them slowly.**

Think of a role as a **uniform with a badge**. The badge grants certain access. Nobody owns the
uniform. When an approved person or service puts it on, they get the badge's access for a limited
time (by default, 1 hour), then it expires.

"Putting on the uniform" is called **assuming the role**. When something assumes a role, a service
called **STS (Security Token Service)** hands it **temporary credentials**: an access key ID, a
secret key, and a session token that stop working after a while.

**Why this matters:** the most common way AWS accounts get hacked is **permanent access keys
leaking**, for example committed to GitHub or left in a config file. A program running on AWS never
needs permanent keys. It assumes a role and AWS gives it fresh temporary credentials
automatically. That's what you'll set up for your web servers.

### 1.3 Policies: the rules, written in JSON

A **policy** is a JSON document (see Module 00, 3.1) that lists permissions. Here's the one you'll
create in this lab, annotated:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadTheWebsiteFiles",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::lab-website-123456789012/*"
    },
    {
      "Sid": "ListTheWebsiteBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::lab-website-123456789012"
    }
  ]
}
```

| Field | Meaning |
|---|---|
| `Version` | The version of the policy *language*. Always write exactly `"2012-10-17"`. It's not today's date and you never change it. |
| `Statement` | A list of rules. |
| `Sid` | "Statement ID", an optional label for humans. |
| `Effect` | `"Allow"` or `"Deny"`. |
| `Action` | One action, or a list. Wildcards work: `"s3:Get*"` means every S3 action starting with Get; `"*"` means every action in AWS. |
| `Resource` | The ARN(s) the rule applies to. `"*"` means every resource. |
| `Condition` | (optional, not used here) Extra requirements, e.g. "only from this IP address" or "only with MFA". |

**Look closely at the two `Resource` lines**, because this is the most common S3 permission mistake:
- `arn:aws:s3:::lab-website-123456789012` is the **bucket itself**. Listing what's in a bucket
  (`s3:ListBucket`) is an action on the *bucket*.
- `arn:aws:s3:::lab-website-123456789012/*` is **every file inside** the bucket. Reading a file
  (`s3:GetObject`) is an action on the *file*.

Mix them up and you get `AccessDenied` even though the policy "looks right".

### 1.4 Where policies attach

- **Identity-based policy**: attached to a user, group, or role. It says *what this identity can do*.
  The policy above is one of these; we'll attach it to the role.
- **Resource-based policy**: attached to a resource (an S3 bucket, a queue, a Lambda function). It
  says *who can use this resource*. It has an extra `Principal` field naming who. You'll meet one in
  Module 08.
- **Trust policy**: a special resource-based policy that **every role has**. It answers "**who is
  allowed to assume this role?**" This is separate from what the role can do.

So **every role has two sides**:

```
                     ┌────────────────────────────────────────────┐
  "Who can wear it?" │ TRUST POLICY                               │
                     │   EC2 servers may assume this role         │
                     │   (and, for testing, my own account)       │
                     ├────────────────────────────────────────────┤
  "What can the      │ PERMISSION POLICIES                        │
   wearer do?"       │   read files in bucket lab-website-...     │
                     │   let Session Manager connect (Module 04)  │
                     └────────────────────────────────────────────┘
                                  lab-web-server-role
```

### 1.5 How IAM decides: the evaluation order

When a request arrives, IAM gathers every policy that applies and decides:

```
1. Is there an explicit "Deny" that matches?        → DENIED. A Deny always wins.
2. Is there an "Allow" that matches?                → ALLOWED
3. Neither?                                         → DENIED (implicit deny)
```

(Large companies add extra layers, such as organization-wide guardrails called SCPs and
permission boundaries, but they follow the same logic: any layer can only take permissions away.)

### 1.6 AWS-managed vs your own policies

- **AWS managed policies** are written and maintained by AWS, e.g. `AdministratorAccess`,
  `AmazonS3ReadOnlyAccess`. Convenient, but often broader than you need.
- **Customer managed policies** are ones you write and can reuse across many identities.
- **Inline policies** are written directly into one user or role and deleted with it. We'll use one
  for the bucket permission because it belongs to exactly one role.

**Least privilege** is the principle of granting only the permissions actually needed, and nothing
more. The role you're about to build follows it: it can read one bucket and do nothing else.

---

## Part 2: Lab (15 min)

Every command runs in **CloudShell**. Paste one block at a time and check the result before moving on.

### Step 1: Look at your own permissions

**Do this:**
```bash
aws iam list-attached-user-policies --user-name admin
```

**What you should see:**
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "AdministratorAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AdministratorAccess"
        }
    ]
}
```

Now read what that policy actually says:
```bash
aws iam get-policy-version \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess \
  --version-id v1 \
  --query PolicyVersion.Document
```

**What you should see:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "*",
            "Resource": "*"
        }
    ]
}
```

**What just happened:** "Administrator" in AWS is just a policy that says *allow every action on
every resource*. There's nothing more to it. The role you're about to build is the opposite: as
narrow as possible.

### Step 2: Create the S3 bucket for your website

S3 gets its own module (05). For now, all you need is this: **S3 stores files**. Files are called
**objects**, and they live in containers called **buckets**. **Bucket names must be unique across
all AWS accounts in the world**, so we put your account ID in the name.

**Do this:**
```bash
BUCKET=lab-website-$ACCOUNT_ID
save BUCKET
aws s3 mb s3://$BUCKET
```

**What you should see:**
```
saved: BUCKET=lab-website-123456789012
make_bucket: lab-website-123456789012
```

**What just happened:** `mb` means "make bucket". `s3://name` is how the CLI writes a bucket address.
The bucket was created in `us-east-1` because that's your configured region.

**If it fails:**
- `BucketAlreadyExists`: someone else in the world has that name. Run `BUCKET=lab-website-$ACCOUNT_ID-2` then `save BUCKET`, and try again.
- The bucket name shows as `lab-website-` with no number: `ACCOUNT_ID` is empty. Redo Module 01, Part 7, Step 4.

### Step 3: Create the web page and upload it

**Do this** (paste the whole block including the `EOF` line):
```bash
cat > index.html <<'EOF'
<!doctype html>
<html>
  <head><title>AWS course</title></head>
  <body style="font-family: sans-serif; text-align: center; margin-top: 15%">
    <h1>Hello from AWS!</h1>
    <p>Served by INSTANCE_ID in AVAILABILITY_ZONE</p>
  </body>
</html>
EOF
aws s3 cp index.html s3://$BUCKET/index.html
aws s3 ls s3://$BUCKET
```

**What you should see:**
```
upload: ./index.html to s3://lab-website-123456789012/index.html
2026-09-27 15:04:11        262 index.html
```

**What just happened:** you wrote a small web page into a file, copied it into the bucket, and listed
the bucket's contents. `INSTANCE_ID` and `AVAILABILITY_ZONE` are placeholders. In Module 04 each web
server will replace them with its own ID and location, so you can see which server answered you.

### Step 4: Write the trust policy ("who can wear the role")

**Do this:**
```bash
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2ServersCanAssumeThisRole",
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    },
    {
      "Sid": "MyAccountCanAssumeThisRoleForTesting",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::$ACCOUNT_ID:root" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
cat trust-policy.json
```

**What you should see:** the JSON printed back, with your real 12-digit account ID where `$ACCOUNT_ID` was.

**What just happened:**
- Notice this heredoc uses `<<EOF` **without quotes**, unlike Step 3. Without quotes, the shell
  replaces `$ACCOUNT_ID` with its value. With quotes (`<<'EOF'`) the text is kept exactly as
  typed. Use unquoted when you *want* variables filled in.
- Statement 1: the **EC2 service** (virtual servers) may assume this role. That's for Module 04.
- Statement 2: **identities in your own account** may assume it, *if their own permissions also
  allow it*. `:root` here means "the account as a whole", not the root user. Your `admin` has
  `AdministratorAccess`, so it qualifies. This lets you test the role in Step 9.

### Step 5: Create the role

**Do this:**
```bash
aws iam create-role \
  --role-name lab-web-server-role \
  --assume-role-policy-document file://trust-policy.json \
  --query Role.Arn --output text
```

**What you should see:**
```
arn:aws:iam::123456789012:role/lab-web-server-role
```

**What just happened:** you created a role with a trust policy and **no permissions yet**. Right now
anyone wearing it could do nothing at all, because of deny by default. `file://` tells the CLI to
read the value from a file instead of the command line.

**If it fails:** `MalformedPolicyDocument` means a JSON typo. Run `cat trust-policy.json` and check the
quotes and commas, or just re-paste Step 4.

### Step 6: Give the role permission to read the bucket

**Do this:**
```bash
cat > read-website-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadTheWebsiteFiles",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::$BUCKET/*"
    },
    {
      "Sid": "ListTheWebsiteBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::$BUCKET"
    }
  ]
}
EOF
aws iam put-role-policy \
  --role-name lab-web-server-role \
  --policy-name read-website-bucket \
  --policy-document file://read-website-policy.json
```

**What you should see:** nothing. Many AWS "write" commands print nothing when they succeed. No
error means it worked.

**What just happened:** `put-role-policy` added an **inline** policy named `read-website-bucket` to the
role. It's exactly the policy from Part 1.3, with your bucket name filled in.

### Step 7: Let Session Manager connect to servers wearing this role

In Module 04 you'll open a terminal *on* your web servers from the browser, using a service called
**Systems Manager Session Manager**. For that to work, the server has to be allowed to talk to
Systems Manager. AWS publishes a managed policy for exactly this.

**Do this:**
```bash
aws iam attach-role-policy \
  --role-name lab-web-server-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

**What you should see:** nothing (success).

**What just happened:** `attach-role-policy` connects an existing **managed** policy to the role. Compare
with Step 6: `put-role-policy` = write an inline policy; `attach-role-policy` = link a managed policy.

### Step 8: Wrap the role in an instance profile

EC2 servers can't use a role directly. They need it wrapped in a container called an **instance
profile** (one role per profile). The web console does this for you silently. The CLI makes you do it
yourself, which is why this step exists.

**Do this:**
```bash
aws iam create-instance-profile --instance-profile-name lab-web-server-profile \
  --query InstanceProfile.Arn --output text
aws iam add-role-to-instance-profile \
  --instance-profile-name lab-web-server-profile \
  --role-name lab-web-server-role
```

**What you should see:** the profile's ARN, e.g. `arn:aws:iam::123456789012:instance-profile/lab-web-server-profile`.

### Step 9: Ask IAM what the role can do (the policy simulator)

Before testing for real, ask IAM to *simulate* decisions without actually doing anything.

**Do this:**
```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::$ACCOUNT_ID:role/lab-web-server-role \
  --action-names s3:GetObject s3:PutObject s3:DeleteObject \
  --resource-arns "arn:aws:s3:::$BUCKET/index.html" \
  --query 'EvaluationResults[].[EvalActionName,EvalDecision]' \
  --output table
```

**What you should see:**
```
-------------------------------------
|      SimulatePrincipalPolicy      |
+------------------+----------------+
|  s3:GetObject    |  allowed       |
|  s3:PutObject    |  implicitDeny  |
|  s3:DeleteObject |  implicitDeny  |
+------------------+----------------+
```

**What just happened:** reading is `allowed` because a statement allows it. Writing and deleting are
`implicitDeny`, meaning *nothing allowed them*, so the default "no" applies. This is least privilege
working.

### Step 10 (optional, 2 min): See that an explicit Deny beats an Allow

**Do this:**
```bash
cat > deny-secrets-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::$BUCKET/secret/*"
    }
  ]
}
EOF
aws iam put-role-policy --role-name lab-web-server-role \
  --policy-name deny-secret-folder --policy-document file://deny-secrets-policy.json

aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::$ACCOUNT_ID:role/lab-web-server-role \
  --action-names s3:GetObject \
  --resource-arns "arn:aws:s3:::$BUCKET/index.html" "arn:aws:s3:::$BUCKET/secret/passwords.txt" \
  --query 'EvaluationResults[].[EvalResourceName,EvalDecision]' --output table
```

**What you should see:**
```
|  arn:aws:s3:::lab-website-.../index.html              |  allowed       |
|  arn:aws:s3:::lab-website-.../secret/passwords.txt    |  explicitDeny  |
```

**What just happened:** the Allow policy covers `bucket/*`, which includes `secret/passwords.txt`.
But the Deny matches too, and **an explicit Deny always wins**. Remove it now so it doesn't affect later
modules:
```bash
aws iam delete-role-policy --role-name lab-web-server-role --policy-name deny-secret-folder
```

### Step 11: Wear the role yourself and test for real

Now assume the role, exactly like an EC2 server will, and try things.

**Do this: assume the role**
```bash
sleep 10
read AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN < <(
  aws sts assume-role \
    --role-arn arn:aws:iam::$ACCOUNT_ID:role/lab-web-server-role \
    --role-session-name testing-the-role \
    --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
    --output text)
export AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
aws sts get-caller-identity --query Arn --output text
```

**What you should see:**
```
arn:aws:sts::123456789012:assumed-role/lab-web-server-role/testing-the-role
```

**What just happened:**
- `sleep 10` waits 10 seconds. IAM changes take a few seconds to spread across AWS (IAM is
  **eventually consistent**). Assuming a brand-new role immediately can fail.
- `aws sts assume-role` asked STS to put on the uniform. STS checked the **trust policy** (your
  account is allowed), then returned three temporary credentials, valid for 1 hour.
- `read ... < <( ... )` split those three values into three variables. `export` made them visible to
  the `aws` command. **The AWS CLI always prefers credentials in these variables over anything else**,
  so from now on in this tab, you *are* the role.
- The "who am I" check confirms it: you're now `assumed-role/lab-web-server-role`.

**Do this: try reading (should work)**
```bash
aws s3 cp s3://$BUCKET/index.html -
```
**What you should see:** your web page's HTML. (`-` as the destination means "print it
to the screen instead of saving a file".)

**Do this: try writing (should fail)**
```bash
echo "hacked" > hacked.txt
aws s3 cp hacked.txt s3://$BUCKET/hacked.txt
```
**What you should see:**
```
upload failed: ./hacked.txt to s3://lab-website-.../hacked.txt An error occurred (AccessDenied)
when calling the PutObject operation: User: arn:aws:sts::123456789012:assumed-role/lab-web-server-role/testing-the-role
is not authorized to perform: s3:PutObject on resource: "arn:aws:s3:::lab-website-.../hacked.txt"
because no identity-based policy allows the s3:PutObject action
```

**Learn to read this message.** Every AWS permission error tells you the same four things:
**who** (`User: ...assumed-role/lab-web-server-role/...`), **which action** (`s3:PutObject`),
**which resource**, and **why** (`no identity-based policy allows`). That's everything you need to
write the fix.

**Do this: try listing all your buckets (should fail)**
```bash
aws s3 ls
```
**What you should see:** `An error occurred (AccessDenied) when calling the ListBuckets operation ...`.
The role can only list *its* bucket, not all of them.

### Step 12: Go back to being admin

**Do this:**
```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
aws sts get-caller-identity --query Arn --output text
rm -f hacked.txt
```

**What you should see:** `arn:aws:iam::123456789012:user/admin`

**What just happened:** `unset` deleted the three variables, so the CLI fell back to CloudShell's
normal credentials, which are you as `admin`.

> **If you ever get strange AccessDenied errors later in this course,** you might still be wearing
> the role in that tab. Run `aws sts get-caller-identity`. If it says `assumed-role`, run the
> `unset` line above.

### Step 13: See it in the console

1. Search `IAM` → **Roles** → search `lab-web` → click **lab-web-server-role**.
2. **Permissions** tab: you'll see `AmazonSSMManagedInstanceCore` (type: AWS managed) and
   `read-website-bucket` (type: Customer inline). Click the `+` next to each to read the JSON.
3. **Trust relationships** tab: your trust policy with its two principals.
4. Search `S3` → click your bucket `lab-website-...` → you'll see `index.html`.

---

## What exists in your account now

| Resource | Name | Used in |
|---|---|---|
| S3 bucket | `lab-website-<account-id>` with `index.html` | Modules 04, 05 |
| IAM role | `lab-web-server-role` | Module 04 |
| Instance profile | `lab-web-server-profile` | Module 04 |

**No cleanup yet.** These are free and Module 04 uses them. Module 04 ends by deleting the role and
profile, and Module 05 deletes the bucket.

---

## Checkpoint

1. A policy allows `s3:GetObject` on `arn:aws:s3:::photos`. Reading `photos/cat.jpg` fails. Why?
2. A user has `AdministratorAccess`, plus a policy with `"Effect": "Deny", "Action": "s3:DeleteObject", "Resource": "*"`. Can they delete a file?
3. Your program running on AWS gets `AccessDenied` for `dynamodb:PutItem`. Do you fix the role's trust policy or its permission policy?
4. Why is a role safer than putting an access key in a config file on the server?
5. The error says `because no identity-based policy allows the s3:PutObject action`. What exactly would you add, and where?

<details><summary>Answers</summary>

1. `GetObject` acts on objects, whose ARN is `arn:aws:s3:::photos/*`. The policy names the bucket itself.
2. No. An explicit Deny beats any Allow.
3. The permission policy. The trust policy only controls who can *assume* the role, and the program has clearly already assumed it.
4. Role credentials are temporary (they expire), rotate automatically, and are never written into files or code where they can leak.
5. A statement `{"Effect": "Allow", "Action": "s3:PutObject", "Resource": "arn:aws:s3:::<bucket>/*"}` in a policy attached to the identity named in the error.
</details>

**Where you are now:** you have a bucket with a web page and a role that can read it. Before you can
launch servers that use the role, they need a network to live in.

**Next: [Module 03 · Networking (VPC)](03-networking-vpc.md)**
