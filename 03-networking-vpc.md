# Module 03 · Networking: Build Your Own Network (VPC)

**Time: about 25 minutes.**
**Before you start:** open CloudShell and check the region is `us-east-1`. Your saved values
(`$ACCOUNT_ID`, `$BUCKET`) load automatically. Check with `cat ~/lab.env`.

**Where you are:** in Module 02 you created a role your web servers will wear. Servers also need a
**network**: somewhere to live, a way to reach the internet, and firewalls controlling who can reach
them. In AWS you build that network yourself.

**What you'll build (and use in Module 04):**

```
Region us-east-1
┌──────────────────────── lab-vpc   10.0.0.0/16 ───────────────────────────┐
│                                                                          │
│      Availability Zone us-east-1a          Availability Zone us-east-1b  │
│  ┌─ lab-public-a   10.0.1.0/24 ─┐        ┌─ lab-public-b   10.0.2.0/24 ─┐ │
│  │  (web server A goes here)    │        │  (web server B goes here)    │ │
│  └──────────────────────────────┘        └──────────────────────────────┘ │
│  ┌─ lab-private-a  10.0.11.0/24 ┐        ┌─ lab-private-b  10.0.12.0/24 ┐ │
│  │  (no internet access)        │        │  (no internet access)        │ │
│  └──────────────────────────────┘        └──────────────────────────────┘ │
│                                                                          │
│  Route table lab-public-rt:   10.0.0.0/16 → local,  0.0.0.0/0 → lab-igw   │
│  Route table lab-private-rt:  10.0.0.0/16 → local   (nothing else)        │
│                                                                          │
│  Security group lab-alb-sg:  allow TCP 80 from anywhere                   │
│  Security group lab-web-sg:  allow TCP 80 only from lab-alb-sg            │
└───────────────────────────────── lab-igw (Internet Gateway) ─────────────┘
                                          │
                                      Internet
```

Everything in this module is **free**.

---

## Part 1: Concepts (10 min read)

### 1.1 VPC: your private network

A **VPC (Virtual Private Cloud)** is a private network inside one AWS region that belongs only to
you. Think of it as an empty office building: nothing inside can talk to the outside world, and
nothing outside can get in, until you add doors and decide who may use them.

When you create a VPC you give it an IP range in CIDR notation (see Module 00, 2.2). We'll use
`10.0.0.0/16`: 65,536 private addresses, all starting with `10.0.`. Every server you put in this VPC
gets a private IP from this range.

> **Every region already has a "default VPC"** (range `172.31.0.0/16`) that AWS created for you, so
> beginners can launch servers without building a network. Real projects build their own, and so
> will you, because building it is how you learn what the parts do.

### 1.2 Subnets: floors of the building

A **subnet** is a smaller slice of the VPC's IP range, for example `10.0.1.0/24` (256 addresses). Key facts:

- **Each subnet lives in exactly one Availability Zone.** This is how you place things in
  different AZs: put them in subnets in different AZs. To survive an AZ failure (Module 01, 1.1),
  you need subnets in at least two AZs.
- AWS reserves **5 addresses in every subnet** for itself (the first four and the last one), so a
  `/24` gives you 251 usable addresses.
- A subnet is called **public** or **private** based on one thing only: its route table (next section).

### 1.3 Route tables: the signs in the hallway

Every subnet is connected to one **route table**. A route table is a list of rules: "traffic going
to *this destination*, send it to *this target*."

```
Destination     Target
10.0.0.0/16     local           ← anything inside the VPC: deliver it directly
0.0.0.0/0       igw-0abc123     ← anything else (the internet): send to the Internet Gateway
```

- The **`local` route** is added automatically and can't be removed. It means every subnet in a VPC
  can reach every other subnet in the same VPC.
- When a destination matches several rules, the **most specific one wins**. `10.0.5.9` matches both
  `10.0.0.0/16` and `0.0.0.0/0`. `/16` is more specific than `/0`, so it goes `local`.
- Each VPC has a **main route table**. Any subnet you don't explicitly connect to another route table
  uses it.

### 1.4 Internet Gateway: the front door

An **Internet Gateway (IGW)** connects a VPC to the internet. You create one and attach it to the
VPC. It does nothing until a route table points `0.0.0.0/0` at it.

**The definition you need to remember:**

> A **public subnet** is a subnet whose route table has `0.0.0.0/0 → Internet Gateway`.
> A **private subnet** is one that doesn't.

That's all there is to it. For a server in a public subnet to actually be reachable from the
internet, it also needs a **public IP address**. We'll tell our public subnets to give every new server
one automatically.

### 1.5 NAT Gateway: a one-way door (we won't build one)

Servers in a **private** subnet can't be reached from the internet, which is good for databases and
internal services. But sometimes they need to reach *out*, for example to download software
updates. A **NAT Gateway** sits in a public subnet and lets private servers make outgoing
connections while blocking all incoming ones.

**We won't create one**, because it costs about $32/month even when idle, plus a fee per GB. It's the
most common surprise charge on AWS bills. You'll still create private subnets so you can see how their
routing differs.

> **In production**, web servers usually live in private subnets, with only the load balancer in
> public subnets, and a NAT Gateway for outgoing traffic. In this course the web servers go in the
> public subnets to avoid the NAT Gateway cost, and we use a firewall rule so that only the load
> balancer can reach them. Module 04 shows why that's still safe.

### 1.6 Security groups: a firewall around each server

A **security group (SG)** is a firewall (see Module 00, 2.6) attached to a server (more precisely, to
its network interface). Rules:

- A new security group allows **no incoming traffic** and **all outgoing traffic**.
- You add **allow** rules only. There are no deny rules. Anything not allowed is blocked.
- Security groups are **stateful**: if a connection is allowed in, the reply is automatically
  allowed back out (and vice versa). You never write rules for replies.
- A rule's source can be an IP range (`0.0.0.0/0`, `203.0.113.5/32`) **or another security group**.

That last point is the most useful trick in AWS networking. Here's what you'll build:

```
Internet ──TCP 80──▶ [ lab-alb-sg ]           rule: allow TCP 80 from 0.0.0.0/0
                     load balancer
                          │
                          └──TCP 80──▶ [ lab-web-sg ]   rule: allow TCP 80 from lab-alb-sg
                                        web servers
```

"Allow from `lab-alb-sg`" means *allow any machine that has the `lab-alb-sg` security group
attached*, whatever its IP address. The load balancer can reach the web servers; nobody on the
internet can reach them directly, even though they have public IPs.

### 1.7 Network ACLs (just know they exist)

A **network ACL (NACL)** is a second, optional firewall that applies to a whole subnet. Unlike
security groups, NACLs are **stateless** (you must allow replies explicitly) and have **deny** rules.
The default NACL allows everything, and most people leave it that way. We will too.

### 1.8 When you can't connect: the checklist

Come back to this list whenever something times out:
1. Is the server in a subnet whose route table has `0.0.0.0/0 → igw-...`?
2. Does the server have a public IP?
3. Does its security group allow the port **from where you're connecting**?
4. Is a NACL blocking it? (Rare if you haven't changed them.)
5. Is the program on the server running and listening on that port?

---

## Part 2: Lab (15 min)

### Step 1: Look at the default VPC

**Do this:**
```bash
aws ec2 describe-vpcs --query 'Vpcs[].[VpcId,CidrBlock,IsDefault]' --output table
```

**What you should see:** one row, `vpc-xxxxxxxx | 172.31.0.0/16 | True`. That's the default VPC AWS made
for you. You'll leave it alone and build your own next to it.

### Step 2: Create your VPC

**Do this:**
```bash
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=lab-vpc}]' \
  --query Vpc.VpcId --output text)
save VPC_ID

aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames '{"Value":true}'
```

**What you should see:** `saved: VPC_ID=vpc-0a1b2c3d4e5f67890`

**What just happened:**
- `create-vpc` created an empty network with the range `10.0.0.0/16`.
- `--tag-specifications` added a **tag**: a label (key `Name`, value `lab-vpc`). The console shows
  the `Name` tag as the resource's display name. You can tag almost everything in AWS, and tags are
  also how companies track which project or team a cost belongs to.
- `modify-vpc-attribute --enable-dns-hostnames` makes AWS give servers in this VPC DNS names, not just IPs.

**If it fails:** `VpcLimitExceeded` means you already have 5 VPCs in this region (the default limit). Delete unused ones in the console (VPC → Your VPCs).

### Step 3: Create four subnets in two Availability Zones

**Do this:**
```bash
PUBLIC_SUBNET_A=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab-public-a}]' \
  --query Subnet.SubnetId --output text)
save PUBLIC_SUBNET_A

PUBLIC_SUBNET_B=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab-public-b}]' \
  --query Subnet.SubnetId --output text)
save PUBLIC_SUBNET_B

PRIVATE_SUBNET_A=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.11.0/24 --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab-private-a}]' \
  --query Subnet.SubnetId --output text)
save PRIVATE_SUBNET_A

PRIVATE_SUBNET_B=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.12.0/24 --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab-private-b}]' \
  --query Subnet.SubnetId --output text)
save PRIVATE_SUBNET_B
```

**What you should see:** four `saved:` lines, each with a `subnet-...` ID.

**What just happened:** you split the VPC into four `/24` slices, two in AZ `us-east-1a` and two in
`us-east-1b`. Each subnet's range must fit inside the VPC's range (`10.0.x.x`) and must not overlap
another subnet. At this moment **all four are private**: the VPC has no Internet Gateway, and every
subnet uses the main route table, which only has the `local` route.

**If it fails:** `InvalidParameterValue ... Value (us-east-1a) for parameter availabilityZone is invalid` means you're not in `us-east-1`. Run `echo $AWS_REGION`, and check Module 01, Part 7, Step 3.

### Step 4: Create the Internet Gateway and attach it

**Do this:**
```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=lab-igw}]' \
  --query InternetGateway.InternetGatewayId --output text)
save IGW_ID

aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
```

**What you should see:** `saved: IGW_ID=igw-...` and nothing else (attach prints nothing on success).

**What just happened:** the VPC now has a door to the internet. **No traffic uses it yet**, because no
route table points to it. Both steps are needed.

### Step 5: Create the public route table and point it at the Internet Gateway

**Do this:**
```bash
PUBLIC_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=lab-public-rt}]' \
  --query RouteTable.RouteTableId --output text)
save PUBLIC_RT

aws ec2 create-route --route-table-id $PUBLIC_RT \
  --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

aws ec2 associate-route-table --route-table-id $PUBLIC_RT --subnet-id $PUBLIC_SUBNET_A \
  --query AssociationState.State --output text
aws ec2 associate-route-table --route-table-id $PUBLIC_RT --subnet-id $PUBLIC_SUBNET_B \
  --query AssociationState.State --output text
```

**What you should see:**
```
saved: PUBLIC_RT=rtb-...
{
    "Return": true
}
associated
associated
```

**What just happened:**
1. You created a new route table. It automatically contains the `local` route.
2. You added the rule `0.0.0.0/0 → Internet Gateway`.
3. You connected ("associated") the two public subnets to it.

**This is the moment `lab-public-a` and `lab-public-b` became public subnets.**

Check the routes:
```bash
aws ec2 describe-route-tables --route-table-ids $PUBLIC_RT \
  --query 'RouteTables[0].Routes[].[DestinationCidrBlock,GatewayId]' --output table
```
**What you should see:**
```
+--------------+------------------------+
|  10.0.0.0/16 |  local                 |
|  0.0.0.0/0   |  igw-0123456789abcdef0 |
+--------------+------------------------+
```

### Step 6: Create the private route table

**Do this:**
```bash
PRIVATE_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=lab-private-rt}]' \
  --query RouteTable.RouteTableId --output text)
save PRIVATE_RT

aws ec2 associate-route-table --route-table-id $PRIVATE_RT --subnet-id $PRIVATE_SUBNET_A \
  --query AssociationState.State --output text
aws ec2 associate-route-table --route-table-id $PRIVATE_RT --subnet-id $PRIVATE_SUBNET_B \
  --query AssociationState.State --output text

aws ec2 describe-route-tables --route-table-ids $PRIVATE_RT \
  --query 'RouteTables[0].Routes[].[DestinationCidrBlock,GatewayId]' --output table
```

**What you should see:** two `associated` lines, then a table with **only** `10.0.0.0/16 | local`.

**What just happened:** the private subnets can reach anything inside the VPC, and nothing outside.
If you had a NAT Gateway, you'd add `0.0.0.0/0 → nat-...` here. That's the only difference between
this table and the public one.

### Step 7: Give servers in the public subnets a public IP automatically

**Do this:**
```bash
aws ec2 modify-subnet-attribute --subnet-id $PUBLIC_SUBNET_A --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $PUBLIC_SUBNET_B --map-public-ip-on-launch
```

**What you should see:** nothing (success).

**What just happened:** any server launched into these two subnets now gets a public IP address as
well as its private one.

Review all four subnets:
```bash
aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID \
  --query 'sort_by(Subnets,&CidrBlock)[].[Tags[?Key==`Name`]|[0].Value,CidrBlock,AvailabilityZone,MapPublicIpOnLaunch]' \
  --output table
```
**What you should see:**
```
+----------------+---------------+-------------+--------+
|  lab-public-a  |  10.0.1.0/24  |  us-east-1a |  True  |
|  lab-private-a |  10.0.11.0/24 |  us-east-1a |  False |
|  lab-private-b |  10.0.12.0/24 |  us-east-1b |  False |
|  lab-public-b  |  10.0.2.0/24  |  us-east-1b |  True  |
+----------------+---------------+-------------+--------+
```
(The order sorts `10.0.11` before `10.0.2` because it sorts as text; that's fine.)

### Step 8: Create the load balancer's security group

**Do this:**
```bash
ALB_SG=$(aws ec2 create-security-group \
  --group-name lab-alb-sg \
  --description "Load balancer: allow HTTP from the internet" \
  --vpc-id $VPC_ID \
  --query GroupId --output text)
save ALB_SG

aws ec2 authorize-security-group-ingress --group-id $ALB_SG \
  --protocol tcp --port 80 --cidr 0.0.0.0/0 \
  --query 'SecurityGroupRules[0].[IpProtocol,FromPort,CidrIpv4]' --output text
```

**What you should see:**
```
saved: ALB_SG=sg-...
tcp	80	0.0.0.0/0
```

**What just happened:** you created a firewall and added one **ingress** (incoming) rule: TCP port 80
from anywhere. Outgoing traffic is allowed by default.

### Step 9: Create the web servers' security group

**Do this:**
```bash
WEB_SG=$(aws ec2 create-security-group \
  --group-name lab-web-sg \
  --description "Web servers: allow HTTP only from the load balancer" \
  --vpc-id $VPC_ID \
  --query GroupId --output text)
save WEB_SG

aws ec2 authorize-security-group-ingress --group-id $WEB_SG \
  --protocol tcp --port 80 --source-group $ALB_SG \
  --query 'SecurityGroupRules[0].[IpProtocol,FromPort,ReferencedGroupInfo.GroupId]' --output text
```

**What you should see:**
```
saved: WEB_SG=sg-...
tcp	80	sg-<the ALB_SG id>
```

**What just happened:** the web servers accept port 80 **only** from machines that carry `lab-alb-sg`.
Notice there's **no rule for port 22 (SSH)**. You'll never need to open it, because in Module 04 you'll
connect through Session Manager, which works over an outgoing connection the server makes itself.

### Step 10: See your network in the console

1. Search `VPC` → **Your VPCs** → click `lab-vpc`.
2. Click the **Resource map** tab. You'll see a diagram: your VPC, the four subnets grouped by AZ, the
   two route tables, and the Internet Gateway, with lines showing which subnet uses which route
   table. **It should match the diagram at the top of this module.** If a line is missing, the
   matching association step didn't work.
3. Left menu → **Security groups** → click `lab-web-sg` → **Inbound rules** tab. The source column
   shows the ID of `lab-alb-sg`.

---

## What exists in your account now

| Resource | Name | Costs money? |
|---|---|---|
| VPC | `lab-vpc` | No |
| Subnets | `lab-public-a/b`, `lab-private-a/b` | No |
| Internet Gateway | `lab-igw` | No |
| Route tables | `lab-public-rt`, `lab-private-rt` | No |
| Security groups | `lab-alb-sg`, `lab-web-sg` | No |
| (from Module 02) | bucket, role, instance profile | No |

**No cleanup yet.** Module 04 uses all of this and ends with a full cleanup.

---

## Checkpoint

1. What single thing makes a subnet "public"?
2. You need servers in two Availability Zones. What's the minimum number of subnets?
3. A security group allows TCP 443 in. Do you need a rule to let the replies out?
4. Why does `lab-web-sg` allow traffic from `lab-alb-sg` instead of from an IP address?
5. A server in `lab-private-a` runs `curl https://example.com`. What happens, and why?
6. Your browser times out connecting to a server. Name three things to check.

<details><summary>Answers</summary>

1. Its route table has a route `0.0.0.0/0 → Internet Gateway`.
2. Two. Each subnet lives in exactly one AZ.
3. No. Security groups are stateful, so replies are allowed automatically.
4. Load balancer IPs change and there can be many. A security-group source means "whatever machines carry that group", which stays correct automatically.
5. It times out. The private route table has no route to the internet (no `0.0.0.0/0` entry), so the traffic has nowhere to go.
6. Any three of: the route table has a route to the IGW; the server has a public IP; the security group allows the port from your IP; the NACL isn't blocking it; the program on the server is actually running.
</details>

**Where you are now:** you have a role (Module 02) and a network (this module). Next, you'll put
servers in it.

**Next: [Module 04 · Compute: EC2 & Load Balancers](04-compute-ec2.md)**
