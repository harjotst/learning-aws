# Final Quiz: Scenario Questions

Answer each one out loud or on paper **before** opening the answer. These are the kinds of
questions you get in system-design interviews and in the AWS Solutions Architect Associate exam.

---

**1.** A startup runs its whole app on a single `t3.large` in a public subnet with the DB
installed on the same box. List the top three changes to make it production-ready.

<details><summary>Answer</summary>

(a) Move the DB to **RDS Multi-AZ** in private subnets, with credentials in Secrets Manager. (b) Run the app in an **Auto Scaling Group across ≥2 AZs** (or ECS Fargate) in private subnets behind an **ALB** in public subnets. (c) Security groups chained ALB → app → DB, plus IaC, CloudWatch alarms, and backups. Bonus: CloudFront + WAF in front.
</details>

**2.** Your Lambda can't connect to your RDS database in a private subnet. The connection times out. What's the likely cause?

<details><summary>Answer</summary>

The Lambda isn't attached to the VPC, *or* it is but the DB's security group doesn't allow the DB port from the Lambda's security group. A timeout (as opposed to "refused" or "auth failed") almost always means routing or security groups.
</details>

**3.** Your Lambda was attached to a VPC to reach RDS and can now no longer call a third-party API on the internet. Why? Fix?

<details><summary>Answer</summary>

VPC-attached Lambdas get no public IP, so private subnets need a **NAT Gateway** route for internet egress. (If it only needs AWS services, VPC endpoints avoid NAT.)
</details>

**4.** Users worldwide complain your static React site on S3 in us-east-1 is slow to load. Fix?

<details><summary>Answer</summary>

Put **CloudFront** in front (edge caching, HTTP/2/3, TLS via ACM) and lock the bucket to CloudFront with **Origin Access Control**, keeping Block Public Access on.
</details>

**5.** A developer committed AWS access keys to a public GitHub repo. What do you do, in order?

<details><summary>Answer</summary>

1) **Deactivate/delete the keys immediately.** 2) Check **CloudTrail** for what they were used for, and look for new IAM users, roles, keys, and instances in *all regions*. 3) Remove anything the attacker created and rotate other secrets. 4) Prevent it happening again: roles instead of keys (OIDC for CI), secret scanning, and GuardDuty.
</details>

**6.** An e-commerce site sends order-confirmation emails in the checkout request. When the email provider is slow, checkouts time out. Redesign.

<details><summary>Answer</summary>

Checkout writes the order and publishes an event (SQS, or SNS/EventBridge → SQS). An email worker (Lambda) consumes it with retries and a DLQ. The worker must be idempotent. Checkout no longer depends on the email provider being up.
</details>

**7.** You need to store user sessions for a web app running on 10 containers. Where?

<details><summary>Answer</summary>

Outside the containers so they stay stateless: **ElastiCache (Valkey/Redis)** for sub-ms latency, or **DynamoDB with TTL**. Not on local disk and not with sticky sessions.
</details>

**8.** Monthly bill: NAT Gateway $300. Most of the traffic is ECS tasks pulling images from ECR and reading from S3. Cheapest fix?

<details><summary>Answer</summary>

Add an **S3 gateway endpoint** (free; ECR image layers are also served from S3) and **interface endpoints** for ECR API/DKR. Traffic stops going through NAT.
</details>

**9.** DynamoDB table with PK=`userId`. A new feature needs "all users in country X who signed up this month." How?

<details><summary>Answer</summary>

Add a **GSI** with PK=`country` and SK=`signupDate`, then Query with `country = X AND signupDate BETWEEN ...`. Don't Scan in a hot path.
</details>

**10.** Your app must survive the loss of an entire AWS region. What changes?

<details><summary>Answer</summary>

Deploy the stack in a second region using IaC, and replicate data: Aurora Global Database / DynamoDB Global Tables / S3 Cross-Region Replication. Use **Route 53** failover or latency routing with health checks. Decide between active-passive and active-active based on RTO/RPO and cost. Multi-AZ alone does *not* cover this.
</details>

**11.** What's the difference between an IAM role's trust policy and its permission policy?

<details><summary>Answer</summary>

The trust policy says *who can assume* the role (e.g. `lambda.amazonaws.com`, another account, a GitHub OIDC identity). The permission policies say *what the role can do* once assumed.
</details>

**12.** A batch analytics job reads 2 TB of CSVs in S3 once a week. Someone proposes loading it all into RDS. Better idea?

<details><summary>Answer</summary>

Query it in place with **Athena** (pay per TB scanned). Convert it to **Parquet** and partition it by date to cut scan cost by 10× or more. Glue Data Catalog holds the schema. Move to Redshift only if the queries become heavy and frequent.
</details>

**13.** Security groups or NACLs: which would you use to allow the app tier to reach the DB tier, and why?

<details><summary>Answer</summary>

Security groups, with the DB SG allowing the port from the *app SG ID*. They're stateful and identity-based (no IP bookkeeping), and they follow instances as they scale. NACLs are for coarse subnet-level blocks.
</details>

**14.** An SQS-triggered Lambda keeps retrying the same message forever and holds up the rest. What's missing?

<details><summary>Answer</summary>

A **dead-letter queue** with a `maxReceiveCount` on the source queue. Also enable partial batch responses (`ReportBatchItemFailures`) so one bad record doesn't fail the whole batch.
</details>

**15.** Name the six pillars of the Well-Architected Framework.

<details><summary>Answer</summary>

Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability.
</details>

---

### Scoring
- **13–15**: you have a solid working model of AWS. Next, build something real with IaC and study for the **Solutions Architect – Associate** certification.
- **9–12**: redo the labs for the modules you missed. Hands-on is what makes it stick.
- **< 9**: reread the "Mental model" sections. Everything else hangs off those.

### Where to go next
1. Rebuild the capstone in **CDK** or **Terraform**, then add a VPC + RDS version with ECS Fargate.
2. [AWS Skill Builder](https://skillbuilder.aws) (free courses and labs) and [AWS Workshops](https://workshops.aws) (guided hands-on builds).
3. Read the AWS docs' *"Best practices"* page for each service you use. They're consistently good.
4. Certification path: **Cloud Practitioner** (optional) → **Solutions Architect Associate** → **Developer / SysOps Associate** → Professional/Specialty.
