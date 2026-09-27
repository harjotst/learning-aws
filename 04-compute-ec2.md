# Module 04 · Compute: Servers (EC2) and Load Balancers

**Time: about 30 minutes** (including a few minutes of waiting for things to start).
**Before you start:** open CloudShell and check you're in `us-east-1`. Run `cat ~/lab.env` and check
you see `BUCKET`, `VPC_ID`, `PUBLIC_SUBNET_A`, `PUBLIC_SUBNET_B`, `ALB_SG`, and `WEB_SG`. If any are
missing, go back to the module that creates them.

**Where you are:** you have a role that can read your web page from S3 (Module 02) and a network with
firewalls (Module 03). Now you'll put servers in it.

**What you'll build:**

```
                         Internet (your browser, CloudShell)
                                     │  http://lab-web-alb-....elb.amazonaws.com
                                     ▼
                 ┌─────── Application Load Balancer (lab-web-alb) ───────┐
                 │   listens on port 80, spread over both public subnets  │
                 └────────────┬───────────────────────────┬───────────────┘
                              │ health checks + traffic   │
                 ┌────────────▼──────────┐   ┌────────────▼──────────┐
                 │ lab-web-a (us-east-1a)│   │ lab-web-b (us-east-1b)│
                 │ EC2 t3.micro          │   │ EC2 t3.micro          │
                 │ wears lab-web-server- │   │ wears lab-web-server- │
                 │ role, reads index.html│   │ role, reads index.html│
                 │ from your S3 bucket   │   │ from your S3 bucket   │
                 └───────────────────────┘   └───────────────────────┘
```

Then you'll **turn off one server and watch the website keep working**. That's the core reason
everything in AWS is built across multiple Availability Zones.

**Cost:** the two servers and the load balancer cost roughly **$0.06 per hour in total**. The
cleanup at the end of this module deletes them.

---

## Part 1: Concepts (8 min read)

### 1.1 EC2: renting a virtual machine

**EC2 (Elastic Compute Cloud)** rents you virtual machines (Module 00, 1.2). AWS calls each one an
**instance**. To launch one, you choose:

| Choice | What it means | What we'll use |
|---|---|---|
| **AMI** (Amazon Machine Image) | The disk image the server starts from: the operating system plus any pre-installed software. | Amazon Linux 2023, AWS's own Linux |
| **Instance type** | How much CPU and memory. | `t3.micro`: 2 virtual CPUs, 1 GB memory, about $0.01/hour |
| **Subnet** | Which network slice, and therefore which AZ. | `lab-public-a` and `lab-public-b` |
| **Security group** | Its firewall. | `lab-web-sg` |
| **IAM instance profile** | The role it wears. | `lab-web-server-profile` (Module 02) |
| **User data** | A script that runs automatically the **first time** it boots. | Installs a web server and fetches your page |
| **Storage** | A virtual disk, called an **EBS volume**, attached to the instance. | The default 8 GB disk |

**Reading an instance type name**, e.g. `m7g.large`:
- `m` is the **family**: `t` = cheap burstable (good for small, spiky loads), `m` = general
  purpose, `c` = more CPU, `r` = more memory, `g`/`p` = GPU.
- `7` is the **generation**: higher is newer and usually better value.
- `g` (optional letters) are **extras**: `g` = AWS's own ARM processors (Graviton, cheaper), `d` = local disk, `n` = faster network.
- `large` is the **size**: `nano < micro < small < medium < large < xlarge < 2xlarge < ...`, each roughly doubling.

**Instance lifecycle:**
```
pending ──▶ running ──▶ stopping ──▶ stopped ──▶ (start again) ──▶ pending ──▶ running
                 └──────────────▶ shutting-down ──▶ terminated (deleted, forever)
```
- **Stop** is like turning a computer off. The disk (EBS volume) and your files remain. You stop
  paying for the CPU but still pay a little for the disk. **The public IP changes** when you start it again.
- **Terminate** is deleting it. The disk is deleted too (by default).

**Ways to pay for EC2** (a common interview question):
| Option | Discount vs normal | Good for |
|---|---|---|
| **On-Demand** | none | Anything short-term or unpredictable (what we're using) |
| **Savings Plans / Reserved Instances** | up to ~72% | Servers you'll run 24/7 for 1-3 years (you commit to spend) |
| **Spot** | up to ~90% | Work that can be interrupted: AWS can take the server back with 2 minutes' warning |

### 1.2 Instance metadata: how a server learns about itself

Every EC2 instance can reach a special address, `http://169.254.169.254`, called the **Instance
Metadata Service (IMDS)**. It answers questions like "what's my instance ID?", "which AZ am I in?", and,
most importantly, **it hands out the temporary credentials of the role the instance wears**. That's how
the AWS CLI on the server gets permission to read your bucket without any stored password.

The current version, **IMDSv2**, requires getting a short-lived token first and then sending it with each
question. You'll require IMDSv2 on your instances because the older version was abused in real
attacks. The user data script below shows the token flow.

### 1.3 Why one server isn't enough: load balancers

If your website runs on one server:
- When that server (or its AZ) fails, the site is down.
- When traffic grows, one server can't keep up.
- If you add a second server, users need a **single address** that sends them to either one.

A **load balancer** solves all three. It's a managed service with a **single DNS name** that receives
all traffic and forwards each request to one of several healthy servers.

An **Application Load Balancer (ALB)** has three parts you configure:

| Part | What it is | Ours |
|---|---|---|
| **Load balancer** | The entry point. You place it in public subnets in **at least two AZs**, with a security group. | `lab-web-alb` in both public subnets, `lab-alb-sg` |
| **Listener** | "Listen on this port/protocol and do this with the request." | Port 80 HTTP → forward to the target group |
| **Target group** | The list of servers to send traffic to, plus a **health check**. | `lab-web-tg` containing both instances |

**Health checks** are the important part. The ALB requests `/` from each server every few seconds. If a
server stops answering with a `200`, the ALB marks it **unhealthy** and stops sending it users. When it
recovers, it's marked **healthy** again and gets traffic back. You'll see this happen.

Other load balancer types: **Network Load Balancer (NLB)** for non-HTTP traffic or extreme performance,
and **Gateway Load Balancer** for network security appliances. The ALB is the one you'll use for websites and APIs.

### 1.4 Auto Scaling (what you'd add next)

In this lab you launch two servers by hand. In production you'd let an **Auto Scaling group (ASG)** do it:
you save the launch settings (AMI, type, security group, role, user data) as a **launch template**, and
tell the ASG "keep at least 2 and at most 10 instances running, spread across these subnets, and
register them in this target group." The ASG then **replaces any instance that fails** and **adds or
removes instances as load changes**. Everything you configure in this lab is exactly what a launch
template and ASG contain.

### 1.5 Other ways to run code on AWS

EC2 gives you the most control and the most work (you manage the operating system). Other options:
- **Containers**: package your app and everything it needs into an **image** (with Docker) and run it
  anywhere. On AWS you store images in **ECR** and run them with **ECS** (AWS's container
  service) or **EKS** (managed Kubernetes). With **Fargate**, you run containers without managing any
  servers at all.
- **Lambda**: upload just a function; AWS runs it when an event happens and bills by the millisecond.
  That's Module 06.

---

## Part 2: Lab (about 22 min)

### Step 1: Find the latest Amazon Linux AMI

AMI IDs are different in every region and change whenever AWS releases an update. Instead of looking one
up by hand, AWS publishes the current ID in a public setting in **Systems Manager Parameter Store**.

**Do this:**
```bash
aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text
```

**What you should see:** an ID like `ami-0abcdef1234567890`.

**What just happened:** you read the latest Amazon Linux 2023 AMI ID for this region. In Step 3 you'll
pass the parameter *name* directly (`resolve:ssm:...`) and EC2 will look up the ID for you, so you never
hard-code it.

### Step 2: Write the user data script

**Do this** (paste the whole block including `EOF`):
```bash
cat > user-data.sh <<'EOF'
#!/bin/bash
# Runs once, as the root user, the first time the server boots.
set -x
dnf install -y httpd
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
aws s3 cp s3://BUCKET_NAME/index.html /tmp/index.html --region us-east-1
sed -e "s/INSTANCE_ID/$INSTANCE_ID/" -e "s/AVAILABILITY_ZONE/$AZ/" /tmp/index.html > /var/www/html/index.html
systemctl enable --now httpd
EOF
sed -i "s/BUCKET_NAME/$BUCKET/" user-data.sh
grep "s3 cp" user-data.sh
```

**What you should see:** `aws s3 cp s3://lab-website-123456789012/index.html /tmp/index.html --region us-east-1`
(with your bucket name, not `BUCKET_NAME`).

**What the script does, line by line** (this runs **on the server**, not in CloudShell):
1. `#!/bin/bash` says "run this file with bash". It must be the very first line.
2. `set -x` prints each command to the log as it runs, which helps debugging (you'll read this log in Step 6).
3. `dnf install -y httpd` installs **Apache httpd**, a web server program. `dnf` is Amazon Linux's
   software installer; `-y` answers "yes" automatically. This needs internet access, which the public
   subnet provides.
4. `TOKEN=$(curl ... -X PUT .../api/token ...)` gets an IMDSv2 token (Part 1.2).
5. The next two lines ask IMDS for this server's **instance ID** and **Availability Zone**.
6. `aws s3 cp s3://.../index.html /tmp/index.html` downloads your page from S3. **No password or key
   anywhere:** the AWS CLI on the server automatically fetches the role's temporary credentials from IMDS.
   This is Module 02 paying off.
7. `sed -e "s/INSTANCE_ID/$INSTANCE_ID/" ...` replaces the placeholders with the real values and writes
   the result to `/var/www/html/index.html`, the folder Apache serves pages from.
8. `systemctl enable --now httpd` starts the web server now (`--now`) and on every future boot (`enable`).

**Why the quoted `<<'EOF'` then `sed -i`?** The script contains `$TOKEN`, `$AZ`, and so on, which must stay
as-is so they run *on the server*. The quoted heredoc keeps them untouched. Then `sed -i` swaps just the
one placeholder, `BUCKET_NAME`, for your real bucket name here in CloudShell.

### Step 3: Launch web server A

**Do this:**
```bash
INSTANCE_A=$(aws ec2 run-instances \
  --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --instance-type t3.micro \
  --subnet-id $PUBLIC_SUBNET_A \
  --security-group-ids $WEB_SG \
  --iam-instance-profile Name=lab-web-server-profile \
  --metadata-options HttpTokens=required \
  --user-data file://user-data.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-web-a}]' \
  --query 'Instances[0].InstanceId' --output text)
save INSTANCE_A
```

**What you should see:** `saved: INSTANCE_A=i-0123456789abcdef0`

**What each option does:**
- `--image-id resolve:ssm:...`: start from the latest Amazon Linux 2023 (Step 1).
- `--instance-type t3.micro`: small and cheap.
- `--subnet-id $PUBLIC_SUBNET_A`: put it in the public subnet in `us-east-1a`. It gets a public IP
  automatically (Module 03, Step 7).
- `--security-group-ids $WEB_SG`: firewall that only allows port 80 from the load balancer.
- `--iam-instance-profile Name=lab-web-server-profile`: wear the role from Module 02.
- `--metadata-options HttpTokens=required`: require IMDSv2.
- `--user-data file://user-data.sh`: run your script on first boot.
- `--tag-specifications ...`: name it `lab-web-a`.

**If it fails:**
- `Value (lab-web-server-profile) for parameter iamInstanceProfile.name is invalid`: the instance profile doesn't exist. Redo Module 02, Step 8.
- An error mentioning the **Free Tier** or the **instance type**: on the Free plan only free-tier-eligible types are allowed. Check you typed `t3.micro` exactly. If it still fails, try `t2.micro`.
- `InvalidSubnetID.NotFound` or `$PUBLIC_SUBNET_A` is empty: run `cat ~/lab.env` and check Module 03 finished.

### Step 4: Wait for it to start, then look at it

**Do this:**
```bash
aws ec2 wait instance-running --instance-ids $INSTANCE_A
aws ec2 describe-instances --instance-ids $INSTANCE_A \
  --query 'Reservations[0].Instances[0].[State.Name,InstanceType,Placement.AvailabilityZone,PrivateIpAddress,PublicIpAddress]' \
  --output table
```

**What you should see** (after 10-30 seconds of apparently nothing happening):
```
+----------+-----------+-------------+------------+----------------+
|  running |  t3.micro |  us-east-1a |  10.0.1.84 |  54.90.123.45  |
+----------+-----------+-------------+------------+----------------+
```

**What just happened:** `aws ec2 wait instance-running` checks every few seconds and returns once the
state is `running`. The instance has a **private IP** from your `10.0.1.0/24` subnet and a **public IP**
from AWS. "Running" means the virtual machine has started. **The user data script takes another 1-2
minutes** to install and start the web server.

Save the public IP:
```bash
PUBLIC_IP_A=$(aws ec2 describe-instances --instance-ids $INSTANCE_A \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
save PUBLIC_IP_A
```

### Step 5: Try to visit it directly (this should fail)

Wait about a minute, then **in your own browser** (a new browser tab, not CloudShell), go to
`http://` followed by the IP address, for example `http://54.90.123.45`. Type `http://` explicitly.

**What you should see:** the page loads forever and eventually says **"This site can't be reached"** or
**"took too long to respond"**.

**Why? Work through the checklist from Module 03, 1.8:**
1. Route to the internet? Yes, `lab-public-a` uses `lab-public-rt` with `0.0.0.0/0 → igw`.
2. Public IP? Yes, you just saw it.
3. **Security group allows port 80 from you? No.** `lab-web-sg` only allows port 80 from `lab-alb-sg`.
   Your laptop doesn't carry that security group.

So the firewall is silently dropping your connection, which is exactly what it's designed to do. A
**timeout** means firewall or routing, as Module 00 said.

### Step 6: Open a terminal on the server with Session Manager

Is the web server actually working? You can check from *inside* the server, without opening any ports.

**First, check the server has registered with Systems Manager:**
```bash
aws ssm describe-instance-information \
  --query 'InstanceInformationList[].[InstanceId,PingStatus]' --output table
```
**What you should see:** your instance ID with `Online`. If the list is empty, wait a minute and try again.
It can take 2-3 minutes after launch.

**Now connect (in the browser):**
1. Search `EC2` → left menu **Instances** → tick the box next to **lab-web-a**.
2. Click **Connect** (top of the page) → choose the **Session Manager** tab → click **Connect**.
3. A new browser tab opens with a black terminal. **You're now typing on the server itself.**

**Do this, inside the Session Manager terminal:**
```bash
curl -s localhost
```
**What you should see:** your web page HTML, including a line like
`<p>Served by i-0123456789abcdef0 in us-east-1a</p>`. The web server works; only the firewall blocked you.

```bash
aws sts get-caller-identity --region us-east-1
```
**What you should see:** `"Arn": "arn:aws:sts::123456789012:assumed-role/lab-web-server-role/i-0123..."`.
The server is wearing your role, and the session name is its instance ID.

```bash
sudo tail -n 15 /var/log/cloud-init-output.log
```
**What you should see:** the last lines of your user data script's output (the `+` lines come from `set -x`),
ending with something about `httpd.service`. **This log is where you look when user data doesn't work.**

When you're done, click **Terminate** (top-right of that tab) to end the session and close the tab.

**If it fails:**
- **Connect** button greyed out / "SSM Agent is not online": wait 2-3 minutes after launch and refresh. It
  needs the instance profile (Module 02, Steps 7-8) and internet access (public subnet).
- `curl localhost` says "Connection refused": user data is still running or failed. Read the log with the
  `tail` command above. If it shows an `AccessDenied` on `s3 cp`, check the role's policy (Module 02, Step 6).

### Step 7: Temporarily allow your own IP (to prove the diagnosis)

1. In your browser, open **https://checkip.amazonaws.com**. It shows your public IP address, e.g. `203.0.113.25`.
2. Back in CloudShell, **replace the example IP with yours**:

```bash
MY_IP=203.0.113.25
aws ec2 authorize-security-group-ingress --group-id $WEB_SG \
  --protocol tcp --port 80 --cidr $MY_IP/32 \
  --query 'SecurityGroupRules[0].CidrIpv4' --output text
```

**What you should see:** `203.0.113.25/32` (your IP).

3. Reload `http://<PUBLIC_IP_A>` in your browser. **The page appears:** "Hello from AWS! Served by i-... in us-east-1a".

**What just happened:** you added a rule allowing port 80 from exactly one address, yours (`/32`). The
change took effect instantly, with no restart. This proves the security group was the only thing in the way.

> **If the browser says it can't make a secure connection:** some browsers try `https://` automatically.
> Make sure the address starts with `http://`, and if the browser offers "Continue to site", click it.

4. **Now remove that rule.** Users should come in through the load balancer, not straight to a server:
```bash
aws ec2 revoke-security-group-ingress --group-id $WEB_SG \
  --protocol tcp --port 80 --cidr $MY_IP/32 --query Return --output text
```
**What you should see:** `True`.

### Step 8: Launch web server B in the other Availability Zone

Same command as Step 3, with a different subnet and name.

**Do this:**
```bash
INSTANCE_B=$(aws ec2 run-instances \
  --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --instance-type t3.micro \
  --subnet-id $PUBLIC_SUBNET_B \
  --security-group-ids $WEB_SG \
  --iam-instance-profile Name=lab-web-server-profile \
  --metadata-options HttpTokens=required \
  --user-data file://user-data.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-web-b}]' \
  --query 'Instances[0].InstanceId' --output text)
save INSTANCE_B
aws ec2 wait instance-running --instance-ids $INSTANCE_B && echo "B is running"
```

**What you should see:** `saved: INSTANCE_B=i-...` and then `B is running`.

### Step 9: Create the target group and register both servers

**Do this:**
```bash
TG_ARN=$(aws elbv2 create-target-group \
  --name lab-web-tg \
  --protocol HTTP --port 80 \
  --vpc-id $VPC_ID \
  --target-type instance \
  --health-check-path / \
  --health-check-interval-seconds 10 \
  --healthy-threshold-count 2 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)
save TG_ARN

aws elbv2 register-targets --target-group-arn $TG_ARN \
  --targets Id=$INSTANCE_A Id=$INSTANCE_B
```

**What you should see:** `saved: TG_ARN=arn:aws:elasticloadbalancing:us-east-1:...:targetgroup/lab-web-tg/...`

**What just happened:**
- `elbv2` is the CLI name for the load balancing service.
- The target group sends HTTP traffic to port 80 on its targets.
- **Health check:** every 10 seconds, request `/`. After 2 successes in a row a target is **healthy**.
  (The default is 5 checks every 30 seconds; we shortened it so you don't have to wait 2.5 minutes.)
- `register-targets` added both instances to the group.

### Step 10: Create the load balancer

**Do this:**
```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name lab-web-alb \
  --type application \
  --scheme internet-facing \
  --subnets $PUBLIC_SUBNET_A $PUBLIC_SUBNET_B \
  --security-groups $ALB_SG \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)
save ALB_ARN

ALB_DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text)
save ALB_DNS

echo "Waiting for the load balancer to become active (usually 2-4 minutes)..."
aws elbv2 wait load-balancer-available --load-balancer-arns $ALB_ARN && echo "ALB is active"
```

**What you should see:** two `saved:` lines, the waiting message, then (after a few minutes) `ALB is active`.

**What just happened:**
- `--type application`: an ALB.
- `--scheme internet-facing`: it gets public addresses. (`internal` would make it reachable only inside the VPC.)
- `--subnets` in **two AZs**: AWS runs part of the load balancer in each, so it survives an AZ failure
  too. An ALB *requires* at least two AZs.
- `--security-groups $ALB_SG`: the firewall allowing port 80 from anywhere. Because it carries
  `lab-alb-sg`, your web servers accept its traffic.
- `ALB_DNS` is the load balancer's permanent address, e.g. `lab-web-alb-1234567890.us-east-1.elb.amazonaws.com`.

**If it fails:** `A load balancer cannot be attached to multiple subnets in the same Availability Zone` or `At least two subnets in two different Availability Zones must be specified` means one of the subnet variables is wrong. Check `cat ~/lab.env`.

### Step 11: Create the listener

The load balancer exists but isn't listening for anything yet.

**Do this:**
```bash
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN \
  --query 'Listeners[0].[Protocol,Port]' --output text
```

**What you should see:** `HTTP	80`

**What just happened:** "When a request arrives on port 80, forward it to `lab-web-tg`." The chain is now complete:
**internet → ALB (port 80) → listener → target group → a healthy instance.**

### Step 12: Check the health of your targets

**Do this:**
```bash
aws elbv2 describe-target-health --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[].[Target.Id,TargetHealth.State,TargetHealth.Description]' \
  --output table
```

**What you should see**, after about 30 seconds:
```
+----------------------+----------+------+
|  i-0aaaaaaaaaaaaaaaa |  healthy |  None|
|  i-0bbbbbbbbbbbbbbbb |  healthy |  None|
+----------------------+----------+------+
```
If you see `initial` (health checks in progress), wait 20 seconds and run it again.

**If it fails:** `unhealthy` with `Health checks failed` or `Request timed out` means either the web server
isn't running on that instance (check with Session Manager, Step 6) or `lab-web-sg` doesn't allow port 80 from
`lab-alb-sg` (Module 03, Step 9).

### Step 13: Use your load-balanced website

**Do this:**
```bash
for i in 1 2 3 4 5 6 7 8; do curl -s http://$ALB_DNS | grep Served; done
echo "Open this in your browser: http://$ALB_DNS"
```

**What you should see:** a mix of both servers:
```
    <p>Served by i-0aaaaaaaaaaaaaaaa in us-east-1a</p>
    <p>Served by i-0bbbbbbbbbbbbbbbb in us-east-1b</p>
    <p>Served by i-0aaaaaaaaaaaaaaaa in us-east-1a</p>
    ...
```

Now open the printed `http://lab-web-alb-....elb.amazonaws.com` address in your browser and refresh a few times.
You'll see the instance ID and AZ change.

**What just happened:** every request went to the same address, and the ALB spread them across both healthy
servers in two different AZs. This is the standard shape of a web application on AWS.

### Step 14: Break something and watch the site survive

Simulate a server failing by stopping server A.

**Do this:**
```bash
aws ec2 stop-instances --instance-ids $INSTANCE_A --query 'StoppingInstances[0].CurrentState.Name' --output text
sleep 30
aws elbv2 describe-target-health --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[].[Target.Id,TargetHealth.State,TargetHealth.Description]' --output table
for i in 1 2 3 4 5 6; do curl -s http://$ALB_DNS | grep Served; done
```

**What you should see:**
- `stopping`
- Server A's health is `unused` or `unhealthy` (`Target is in the stopped state`, or health checks failed). Server B is `healthy`.
- **Every** response now says `Served by <server B> in us-east-1b`.

**What just happened:** the ALB noticed server A wasn't answering health checks and stopped sending it users.
Your website kept working the whole time. If you'd been refreshing during the first few seconds, you might
have seen one or two errors before the ALB noticed. That's why health checks are fast.

**Bring server A back:**
```bash
aws ec2 start-instances --instance-ids $INSTANCE_A --query 'StartingInstances[0].CurrentState.Name' --output text
aws ec2 wait instance-running --instance-ids $INSTANCE_A
aws ec2 describe-instances --instance-ids $INSTANCE_A \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text
echo "Old public IP was: $PUBLIC_IP_A"
sleep 45
for i in 1 2 3 4 5 6; do curl -s http://$ALB_DNS | grep Served; done
```

**What you should see:**
- The new public IP is **different** from the old one. Stop/start gives a new public IP.
- After about 45 seconds, both servers answer again. The web server started automatically because of
  `systemctl enable`, and the page survived because it's stored on the EBS disk.

**The lesson:** users never used the server IPs. They used the ALB's DNS name, which didn't change. That's
another reason to put a load balancer in front of servers.

### Step 15: See it all in the console

1. **EC2 → Instances**: `lab-web-a` and `lab-web-b`, each in a different AZ. Click one and look at the
   **Details**, **Security**, and **Networking** tabs. You'll recognize every setting.
2. **EC2 → Load Balancers** (left menu, near the bottom) → click `lab-web-alb` → **Resource map** tab: listener → target group → two targets, all green.
3. **EC2 → Target Groups** → `lab-web-tg` → **Targets** tab: health status of each.

---

## Part 3: Cleanup (5 min). Do this now.

The servers and the load balancer charge by the hour. The network and IAM pieces are free, but you won't
need them again, so delete everything from Modules 02-04 **except the S3 bucket** (Module 05 uses it).

Delete in this order. AWS won't delete things that other things still depend on.

### Cleanup 1: The load balancer and target group
```bash
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
aws elbv2 wait load-balancers-deleted --load-balancer-arns $ALB_ARN && echo "ALB deleted"
aws elbv2 delete-target-group --target-group-arn $TG_ARN && echo "Target group deleted"
```
**What you should see:** `ALB deleted` then `Target group deleted`. (Deleting the ALB also deletes its listener.)

### Cleanup 2: The instances
```bash
aws ec2 terminate-instances --instance-ids $INSTANCE_A $INSTANCE_B \
  --query 'TerminatingInstances[].CurrentState.Name' --output text
aws ec2 wait instance-terminated --instance-ids $INSTANCE_A $INSTANCE_B && echo "Instances terminated"
```
**What you should see:** `shutting-down shutting-down`, then about a minute later `Instances terminated`.

### Cleanup 3: The security groups
```bash
aws ec2 delete-security-group --group-id $WEB_SG && echo "web SG deleted"
until aws ec2 delete-security-group --group-id $ALB_SG 2>/dev/null; do
  echo "Load balancer's network interfaces still releasing; retrying in 15s..."; sleep 15
done; echo "ALB SG deleted"
```
**What just happened:** `lab-web-sg` goes first because its rule refers to `lab-alb-sg`. The `until ... done`
loop retries deleting `lab-alb-sg` every 15 seconds, because the load balancer's network connections can take a
minute or two to disappear after it's deleted. If you see a `DependencyViolation` error on the first line, wait a minute and run it again.

### Cleanup 4: The network
```bash
for SUBNET in $PUBLIC_SUBNET_A $PUBLIC_SUBNET_B $PRIVATE_SUBNET_A $PRIVATE_SUBNET_B; do
  aws ec2 delete-subnet --subnet-id $SUBNET && echo "deleted $SUBNET"
done
aws ec2 delete-route-table --route-table-id $PUBLIC_RT && echo "public route table deleted"
aws ec2 delete-route-table --route-table-id $PRIVATE_RT && echo "private route table deleted"
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID && echo "IGW deleted"
aws ec2 delete-vpc --vpc-id $VPC_ID && echo "VPC deleted"
```
**What you should see:** a `deleted`/`deleted` line for each. Deleting a subnet automatically removes its route-table association,
which is why the route tables can be deleted right after.

### Cleanup 5: The IAM role and instance profile
```bash
aws iam remove-role-from-instance-profile --instance-profile-name lab-web-server-profile --role-name lab-web-server-role
aws iam delete-instance-profile --instance-profile-name lab-web-server-profile
aws iam detach-role-policy --role-name lab-web-server-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role-policy --role-name lab-web-server-role --policy-name read-website-bucket
aws iam delete-role --role-name lab-web-server-role && echo "Role deleted"
```
A role can't be deleted while it's inside an instance profile or has policies, so those go first.

### Cleanup 6: Verify
```bash
echo "Instances (should be empty or 'terminated'):"
aws ec2 describe-instances --filters Name=tag:Name,Values=lab-web-a,lab-web-b \
  --query 'Reservations[].Instances[].State.Name' --output text
echo "Load balancers named lab-*:"
aws elbv2 describe-load-balancers --query "LoadBalancers[?starts_with(LoadBalancerName,'lab-')].LoadBalancerName" --output text
echo "VPCs named lab-vpc:"
aws ec2 describe-vpcs --filters Name=tag:Name,Values=lab-vpc --query 'Vpcs[].VpcId' --output text
```
**What you should see:** only `terminated` (or nothing) for instances, and nothing for the other two.

**If a cleanup command fails:** `DependencyViolation` means something still uses it. Wait a minute and run that
step again. `NotFound` means it's already deleted, which is fine.

---

## Checkpoint

1. What's the difference between stopping and terminating an instance?
2. How did the web server get permission to read from S3 without any password?
3. You open `http://<server-public-ip>` and it times out, but `curl localhost` on the server works. What's wrong?
4. Why must an ALB be in at least two AZs, and why did you put servers in two AZs?
5. Server A stopped. How did the ALB know not to send users there?
6. Which log file do you read when user data didn't do what you expected?
7. What would an Auto Scaling group have done when server A stopped?

<details><summary>Answers</summary>

1. Stop turns it off and keeps the disk (you can start it again, and it gets a new public IP). Terminate deletes it, and by default its disk too.
2. The instance wore `lab-web-server-role` through its instance profile. The AWS CLI on the server got temporary credentials for that role from the instance metadata service (IMDS).
3. A firewall (security group) isn't allowing port 80 from your IP. The server and web program are fine.
4. So that losing one AZ (a data center problem) doesn't take the site down. The ALB runs in each AZ, and there are healthy servers in the surviving one.
5. Health checks: the ALB requests `/` every 10 seconds, and when A stopped answering it was marked unhealthy and removed from rotation.
6. `/var/log/cloud-init-output.log` on the instance.
7. It would have noticed the unhealthy instance, terminated it, and launched a replacement from the launch template, with no human involved.
</details>

**Where you are now:** you've built the classic highly available web architecture by hand, and taken it
apart again. Your S3 bucket still exists. Next, you'll learn what S3 can really do, and meet a database.

**Next: [Module 05 · Storage & Databases](05-storage-databases.md)**
