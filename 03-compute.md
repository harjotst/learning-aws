# 03 · Compute: EC2, Containers, Lambda (30 min)

## Mental model

The compute options sit on a spectrum from "you manage more, you control more" to "you manage
less, AWS runs it for you":

```
More control / more ops                                        Less ops / more constraints
◀────────────────────────────────────────────────────────────────────────────────────────▶
   EC2              ECS or EKS on EC2        ECS/EKS on Fargate          Lambda
   (virtual         (you run containers,     (containers, no servers    (functions, run on
    machines)        on servers you manage)   to manage)                 events, pay per ms)
```

**Default choice for a new app in 2026:** Lambda for event-driven or spiky workloads and small
APIs. **ECS on Fargate** for long-running services and anything already in a container. EC2
when you need something the others can't give you (GPUs, special OS/kernel needs, very steady
high load where it's cheaper, licensed software). EKS when you're committed to Kubernetes.

---

## EC2 (Elastic Compute Cloud)

A virtual machine. You choose:

- **AMI** (Amazon Machine Image): the disk image / OS, e.g. Amazon Linux 2023, Ubuntu, or your own baked image.
- **Instance type**: `family` `generation` `attributes` `.size`, e.g. **`m7g.large`**:
  - `t` burstable (cheap, CPU credits), `m` general purpose, `c` compute, `r` memory, `g`/`p` GPU, `i` storage
  - the `g` in `m7g` means **Graviton** (AWS's ARM chips, ~20% cheaper for the same work)
- **Storage**: **EBS** volumes (network block storage, persists independently of the instance) or
  *instance store* (local NVMe, fast, **wiped when the instance stops**).
- **Networking**: VPC, subnet, security groups (module 02).
- **IAM role** via an *instance profile*: how code on the box gets AWS credentials.
- **User data**: a script that runs on first boot.

### Pricing models (a common interview topic)
| Model | Discount | Use for |
|---|---|---|
| **On-Demand** | 0% | Unpredictable, short-term |
| **Savings Plans** / **Reserved Instances** | up to ~72% | Steady baseline load; commit $/hr for 1 or 3 years |
| **Spot** | up to ~90% | Fault-tolerant work (batch, CI, stateless workers). AWS can reclaim it with **2 minutes' notice** |

### Scaling & availability: the classic pattern
```
               Route 53 ──▶ Application Load Balancer (public subnets, 2+ AZs)
                                      │ health checks
                     ┌────────────────┴────────────────┐
               EC2 (AZ a)                         EC2 (AZ b)        ← Auto Scaling Group
               private subnet                     private subnet      min=2, desired=2, max=10
                                                                      scale on CPU / request count
```
- **Launch Template**: the recipe (AMI, type, SG, user data, role).
- **Auto Scaling Group (ASG)**: keeps N healthy instances running across AZs, replaces failed
  ones, and scales on metrics. *Treat instances as cattle, not pets.*
- **Elastic Load Balancing**:
  - **ALB** (Layer 7, HTTP/HTTPS): path/host routing, TLS termination, targets can be instances, IPs, Lambda. The one you'll use 90% of the time.
  - **NLB** (Layer 4, TCP/UDP): extreme throughput, static IPs, non-HTTP protocols.
  - **GWLB**: for inserting third-party network appliances. Rare.

### Access without SSH
Use **SSM Session Manager**: give the instance the `AmazonSSMManagedInstanceCore` policy on its
role and you get a shell via `aws ssm start-session`. No port 22, no key pairs, and every
session is logged.

### Instance metadata (IMDS)
From inside an instance, `http://169.254.169.254` tells it about itself (ID, AZ, role
credentials). Always require **IMDSv2** (token-based). IMDSv1 was the vector in the 2019 Capital
One breach via SSRF.

---

## Containers: ECS, EKS, Fargate, ECR

- **ECR**: a private Docker registry. `docker push` your images here.
- **ECS**: AWS's own container orchestrator. Simpler than Kubernetes and tightly integrated.
  - **Task definition**: like a docker-compose service (image, CPU/memory, env, ports, **task role**).
  - **Task**: a running instance of a task definition.
  - **Service**: keeps N tasks running, registers them with an ALB, and does rolling deploys.
  - **Cluster**: logical grouping.
- **EKS**: managed Kubernetes control plane. Choose it for the k8s ecosystem/portability. More power, more complexity.
- **Fargate**: the *capacity* option. Tasks/pods run without you managing EC2 instances. You pay
  per vCPU-second and GB-second.

Rule of thumb: **ECS + Fargate** unless you have a reason for Kubernetes.

---

## Lambda

You upload a function. AWS runs it **in response to events** and bills per request plus per
millisecond of execution × memory.

```python
# handler.py
def handler(event, context):
    # event: the trigger's payload (HTTP request, S3 notification, SQS batch...)
    return {"statusCode": 200, "body": "hi"}
```

Key facts:
- **Timeout max 15 minutes.** Memory 128 MB to 10 GB, and **CPU scales with memory**. If it's
  slow, try giving it more memory; it can end up cheaper because it finishes sooner.
- **Cold start**: the first invocation on a new execution environment initializes the runtime
  and your init code (~100 ms to 1 s+). Code outside the handler runs once per environment, so
  put SDK clients and DB connections there. Mitigations: provisioned concurrency, SnapStart
  (Java/Python/.NET), smaller packages.
- **Concurrency**: each concurrent request gets its own environment. The default account limit
  is 1,000 concurrent per region (can be raised). **Reserved concurrency** caps or guarantees a function's share.
- **Invocation types**: *synchronous* (API Gateway waits for the response), *asynchronous* (S3,
  SNS, EventBridge: Lambda queues it and retries twice), *poll-based* (SQS, Kinesis, DynamoDB
  Streams: Lambda polls and sends batches).
- **Stateless**: `/tmp` is scratch space (512 MB default, up to 10 GB) that may or may not persist between invocations. Store state in S3/DynamoDB.
- Can run in a VPC to reach private resources like RDS.
- Package as a zip (≤250 MB unzipped) or a container image (≤10 GB).

Lambda is covered hands-on in modules 05 and 07.

---

## Lab: launch a web server with user data (10 min)

Uses the VPC from module 02 (`$SUBNET`, `$SG`). If you cleaned it up, use your **default VPC**
instead: drop `--subnet-id`, and create an SG in the default VPC that allows port 80.

```bash
export AWS_REGION=us-east-1

cat > userdata.sh <<'EOF'
#!/bin/bash
dnf install -y httpd
TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
echo "<h1>Hello from $ID in $AZ</h1>" > /var/www/html/index.html
systemctl enable --now httpd
EOF

INSTANCE=$(aws ec2 run-instances \
  --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --instance-type t3.micro \
  --subnet-id $SUBNET --security-group-ids $SG \
  --metadata-options HttpTokens=required \
  --user-data file://userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-web}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $INSTANCE
IP=$(aws ec2 describe-instances --instance-ids $INSTANCE \
      --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo "http://$IP"
sleep 60 && curl http://$IP     # user data takes ~a minute: <h1>Hello from i-0... in us-east-1a</h1>
```

Notes:
- `resolve:ssm:` looks up the latest Amazon Linux AMI ID from a public SSM parameter, so you never hard-code AMI IDs (they differ per region).
- `HttpTokens=required` enforces IMDSv2. The user data uses the token flow.
- If `curl` hangs, work through the debugging checklist in module 02.

**Try it:** `aws ec2 stop-instances --instance-ids $INSTANCE`, then start it again. The public
IP changes (use an **Elastic IP** or, better, a load balancer for a stable address). The EBS
root volume and your file persist.

### Cleanup
```bash
aws ec2 terminate-instances --instance-ids $INSTANCE
aws ec2 wait instance-terminated --instance-ids $INSTANCE
rm -f userdata.sh
# then run the VPC cleanup from module 02
```

## Check yourself

1. Nightly batch job, 40 min, can be restarted if interrupted. Cheapest compute?
2. Image resizing triggered when a file lands in S3, a few seconds each, very spiky volume. What do you use?
3. Your Lambda takes 3 s at 128 MB. Why might 1024 MB be both faster *and* cheaper?
4. Your app on EC2 needs to read from S3. How does it get credentials?
5. Stateless web API in a Docker image, steady traffic, the team doesn't know Kubernetes. What do you use?

<details><summary>Answers</summary>

1. EC2 **Spot** (or Fargate Spot / AWS Batch on Spot). Lambda's 15-minute limit rules it out.
2. Lambda with an S3 event trigger.
3. CPU scales with memory. If it runs 8× faster at 8× the memory, the cost is the same, and it's often better than linear for CPU-bound work. (Tools like AWS Lambda Power Tuning measure this.)
4. An IAM role attached via an instance profile. The SDK gets temporary credentials from IMDS automatically.
5. ECS on Fargate behind an ALB.
</details>

**Next → [04 Storage & Databases](04-storage-databases.md)**
