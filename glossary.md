# Glossary

Every term used in the course, in plain words, with the module where it's explained in full.

| Term | Meaning | Module |
|---|---|---|
| **Access key** | A permanent ID + secret pair that lets a program act as an IAM user. Avoid them: they leak. Use roles instead. | 02 |
| **ACM** | AWS Certificate Manager. Free TLS certificates for `https://`. | 07 |
| **Alarm (CloudWatch)** | Watches one metric; switches between `OK`, `ALARM`, and `INSUFFICIENT_DATA`; can notify SNS. | 07 |
| **ALB** | Application Load Balancer. Spreads HTTP(S) traffic across healthy targets in multiple AZs. | 04 |
| **AMI** | Amazon Machine Image. The disk image (OS + software) an EC2 instance starts from. | 04 |
| **API** | A set of operations a program can call over the network. Everything in AWS is an API call. | 00 |
| **API Gateway** | Gives Lambda functions (and other backends) a public HTTPS URL. | 08 |
| **ARN** | Amazon Resource Name: `arn:aws:service:region:account:resource`. A unique ID for any resource. | 00 |
| **Assume a role** | Temporarily take on a role's permissions and receive temporary credentials from STS. | 02 |
| **Aurora** | AWS's high-performance MySQL/PostgreSQL-compatible database. | 05 |
| **Auto Scaling group (ASG)** | Keeps a set number of EC2 instances healthy, replacing failed ones and scaling with load. | 04 |
| **Availability Zone (AZ)** | An isolated data center location inside a region. Use at least two for high availability. | 01 |
| **Bucket** | A container for objects in S3. Names are globally unique. | 02, 05 |
| **Bucket policy** | A resource-based policy attached to an S3 bucket. | 05 |
| **Change set** | CloudFormation's preview of what an update will add, modify, remove, or replace. | 07 |
| **CIDR** | Notation for an IP range, e.g. `10.0.0.0/16` (65,536 addresses). | 00, 03 |
| **CLI** | Command Line Interface. The `aws` command. | 00, 01 |
| **CloudFormation** | AWS's infrastructure-as-code service. Templates in, stacks out. | 07 |
| **CloudFront** | AWS's CDN: caches content in locations worldwide, close to users. | 05 |
| **CloudShell** | A browser-based terminal in the AWS console, already logged in. | 01 |
| **CloudTrail** | The audit log of every API call in your account: who did what, when. | 07 |
| **CloudWatch** | Metrics, logs, alarms, and dashboards. | 06, 07 |
| **Cold start** | Starting a new copy of a Lambda function: loading the runtime and running code outside the handler. | 06 |
| **Concurrency** | How many copies of a Lambda function are running at the same time. | 06 |
| **Condition expression** | A DynamoDB rule that makes a write happen only if a condition holds, e.g. `attribute_not_exists(PK)`. | 05 |
| **Dead-letter queue (DLQ)** | A queue where messages go after failing too many times, so they don't block processing. | 06 |
| **Decoupling** | Connecting parts of a system through queues/events so each can fail, scale, and change independently. | 06 |
| **Delete marker** | In a versioned S3 bucket, a placeholder that hides a "deleted" object. Remove it to restore the object. | 05 |
| **Dimension** | A name/value pair that identifies a metric, e.g. `FunctionName=lab-order-worker`. | 07 |
| **DNS** | Translates names (`example.com`) into IP addresses. | 00 |
| **Drift** | When real resources no longer match the CloudFormation template because of manual changes. | 07 |
| **DynamoDB** | AWS's serverless key-value/NoSQL database. | 05 |
| **EBS** | Elastic Block Store. A virtual disk attached to an EC2 instance. | 04, 05 |
| **EC2** | Elastic Compute Cloud. Rent virtual machines (instances). | 04 |
| **ECS / EKS / Fargate** | Run containers: ECS is AWS's own orchestrator, EKS is managed Kubernetes, Fargate runs containers without servers to manage. | 04 |
| **EFS** | Elastic File System. A shared network filesystem. | 05 |
| **Event** | Data describing something that happened, passed to a Lambda function. | 06 |
| **Event source mapping** | The link that makes Lambda poll a queue or stream and invoke a function with its messages. | 06 |
| **EventBridge** | An event bus that routes events by content, plus scheduled (cron) events. | 06 |
| **Explicit deny** | A policy statement with `"Effect": "Deny"`. Always overrides any Allow. | 02 |
| **GSI** | Global Secondary Index. A DynamoDB index with a different key, for a different access pattern. | 05 |
| **GuardDuty** | Threat detection that analyzes your account's activity for signs of attack. | 07 |
| **Handler** | The function Lambda calls, e.g. `lambda_function.handler`. | 06 |
| **Health check** | A load balancer's regular test request to each target, to decide if it's healthy. | 04 |
| **Heredoc** | Shell syntax (`cat > file <<'EOF' ... EOF`) for writing multi-line text into a file. | 00 |
| **IaC** | Infrastructure as Code. Define infrastructure in files and let a tool create it. | 07 |
| **IAM** | Identity and Access Management. Decides who may do what. Global, not regional. | 02 |
| **Idempotent** | Doing it twice has the same effect as doing it once. Needed for queue consumers. | 06 |
| **IMDS** | Instance Metadata Service at `169.254.169.254`. Tells an instance about itself and provides its role credentials. Use IMDSv2. | 04 |
| **Implicit deny** | The default answer when no policy allows an action. | 02 |
| **Inline policy** | A policy embedded directly in one user or role. | 02 |
| **Instance** | One EC2 virtual machine. | 04 |
| **Instance profile** | The container that attaches an IAM role to an EC2 instance. | 02, 04 |
| **Instance type** | An instance's size and hardware, e.g. `t3.micro`. | 04 |
| **Internet Gateway (IGW)** | Connects a VPC to the internet. | 03 |
| **JMESPath** | The query language used by the CLI's `--query` option. | 00 |
| **JSON / YAML** | Text formats for structured data. | 00 |
| **KMS** | Key Management Service. Creates and controls encryption keys. | 07 |
| **Lambda** | Runs your function in response to events, without servers. Billed per request and per millisecond. | 06 |
| **Least privilege** | Grant only the permissions actually needed. | 02 |
| **Lifecycle rule** | An S3 rule that moves or deletes objects automatically based on age. | 05 |
| **Listener** | The part of a load balancer that accepts connections on a port and decides where to send them. | 04 |
| **Managed policy** | A reusable policy, written by AWS (`AdministratorAccess`) or by you. | 02 |
| **Metric** | A number tracked over time in CloudWatch, e.g. Lambda `Errors`. | 07 |
| **MFA** | Multi-factor authentication: a code from your phone in addition to a password. | 01 |
| **Multi-AZ (RDS)** | A standby database copy in another AZ with automatic failover. | 05 |
| **NACL** | Network ACL. An optional, stateless firewall on a subnet. | 03 |
| **NAT Gateway** | Lets private subnets make outgoing internet connections. Costs money even when idle. | 03 |
| **Object** | A file stored in S3, identified by its key. | 05 |
| **On-demand** | Pay by usage with no commitment (EC2 pricing, or DynamoDB capacity mode). | 04, 05 |
| **Partition key / sort key** | The parts of a DynamoDB primary key. The partition key decides where an item lives; the sort key orders items within it. | 05 |
| **Policy** | A JSON document of Allow/Deny statements. | 02 |
| **Port** | A number identifying which program on a machine a connection is for (80 = HTTP, 443 = HTTPS). | 00 |
| **Prefix** | The part of an S3 key before a `/`, displayed like a folder. | 05 |
| **Presigned URL** | A time-limited link that grants access to one S3 object. | 05 |
| **Principal** | Who is making a request: a user, role, or service. | 02 |
| **Private / public subnet** | Public: its route table sends `0.0.0.0/0` to an Internet Gateway. Private: it doesn't. | 03 |
| **Query vs Scan** | DynamoDB: Query reads one partition (fast); Scan reads the whole table (slow and costly at scale). | 05 |
| **RDS** | Relational Database Service: managed PostgreSQL, MySQL, and others. | 05 |
| **Read replica** | A read-only copy of a database, for handling more read traffic. | 05 |
| **Region** | A geographic area containing multiple AZs. Most resources live in one region. | 01 |
| **Replacement** | When a CloudFormation update must delete and recreate a resource (dangerous for data). | 07 |
| **Resource-based policy** | A policy attached to a resource (bucket, queue, function) naming who may use it. | 02, 05, 08 |
| **Role** | An IAM identity without permanent credentials that is assumed temporarily. | 02 |
| **Root user** | The account owner's login. Can do everything. Lock it away with MFA. | 01 |
| **Route table** | Rules deciding where network traffic from a subnet goes. | 03 |
| **S3** | Simple Storage Service. Object storage. | 02, 05 |
| **Secrets Manager** | Stores and rotates passwords and API keys. | 07 |
| **Security group** | A stateful firewall attached to an instance or other network interface. Allow rules only. | 03 |
| **Serverless** | You don't manage servers; you pay per use. Lambda, S3, DynamoDB, SQS, API Gateway. | 06 |
| **SNS** | Simple Notification Service. Publish one message to many subscribers (queues, functions, email). | 06, 07 |
| **Spot instance** | Spare EC2 capacity at up to ~90% off that AWS can reclaim with 2 minutes' notice. | 04 |
| **SQS** | Simple Queue Service. Managed message queues. | 06 |
| **SSM Session Manager** | Open a terminal on an instance from the browser, without SSH or open ports. | 04 |
| **Stack** | One deployed copy of a CloudFormation template. | 07 |
| **Stateful / stateless firewall** | Stateful (security groups) allows replies automatically; stateless (NACLs) needs rules for both directions. | 03 |
| **Step Functions** | Runs multi-step workflows with branching, waits, and retries. | 06 |
| **STS** | Security Token Service. Issues temporary credentials when a role is assumed. | 02 |
| **Subnet** | A slice of a VPC's IP range in one AZ. | 03 |
| **Tag** | A key/value label on a resource, e.g. `Name=lab-vpc`. Used for naming and cost tracking. | 03 |
| **Target group** | The set of targets (e.g. instances) a load balancer sends traffic to, with a health check. | 04 |
| **Template** | A CloudFormation YAML/JSON file describing resources. | 07 |
| **Timeout (network)** | A connection that never gets an answer, usually a firewall or routing problem. | 00, 03 |
| **Trust policy** | The part of a role saying who may assume it. | 02 |
| **User data** | A script an EC2 instance runs on first boot. | 04 |
| **Versioning** | S3 keeps every version of an object, so overwrites and deletes can be undone. | 05 |
| **Visibility timeout** | How long SQS hides a received message before it can be received again. | 06 |
| **VPC** | Virtual Private Cloud. Your private network in a region. | 03 |
| **VPC endpoint** | A private path from a VPC to an AWS service, without the internet or a NAT Gateway. | 03 |
| **WAF** | Web Application Firewall. Filters malicious web traffic. | 07 |
| **Well-Architected Framework** | AWS's six pillars for judging designs: operational excellence, security, reliability, performance efficiency, cost optimization, sustainability. | 07 |
