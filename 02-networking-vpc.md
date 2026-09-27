# 02 · Networking: VPC (30 min)

Networking is where most "it's deployed but I can't reach it" hours go. If you understand this
page you can debug most connectivity problems.

## Mental model

A **VPC** (Virtual Private Cloud) is **your own private network inside a region**. Nothing gets in
or out unless you build a path for it *and* allow it through the firewalls.

Connectivity always comes down to two questions:
1. **Routing**: is there a route to the destination? (route tables, gateways)
2. **Filtering**: is the traffic allowed? (security groups, NACLs)

If either one says no, the connection times out.

## The picture

```
Region us-east-1
┌──────────────────────────── VPC 10.0.0.0/16 ─────────────────────────────┐
│                                                                          │
│   AZ a                               AZ b                                │
│  ┌─ Public subnet 10.0.1.0/24 ─┐    ┌─ Public subnet 10.0.2.0/24 ─┐      │
│  │  Load balancer, NAT GW      │    │  Load balancer node         │      │
│  └─────────────────────────────┘    └─────────────────────────────┘      │
│  ┌─ Private subnet 10.0.11.0/24┐    ┌─ Private subnet 10.0.12.0/24┐      │
│  │  App servers / containers   │    │  App servers / containers   │      │
│  └─────────────────────────────┘    └─────────────────────────────┘      │
│  ┌─ DB subnet 10.0.21.0/24 ────┐    ┌─ DB subnet 10.0.22.0/24 ────┐      │
│  │  RDS primary                │    │  RDS standby                │      │
│  └─────────────────────────────┘    └─────────────────────────────┘      │
│                                                                          │
└──────────────── Internet Gateway (IGW) ──────────────────────────────────┘
                              │
                          Internet
```

This three-tier, two-AZ layout is the reference architecture. Learn to draw it from memory.

## The pieces

| Piece | What it does |
|---|---|
| **CIDR block** | The VPC's IP range, e.g. `10.0.0.0/16` (65,536 addresses). Pick one that won't overlap networks you'll connect to later. |
| **Subnet** | A slice of the VPC's range that lives in **exactly one AZ**. AWS reserves 5 IPs in each. |
| **Route table** | Rules that say where traffic goes. Every subnet is associated with one. |
| **Internet Gateway (IGW)** | The VPC's door to the internet, used in both directions. |
| **NAT Gateway** | Lets **private** subnets make *outbound* connections (e.g. `pip install`, calling Stripe) while nothing can connect in. It lives in a public subnet. **It costs money by the hour and per GB (~$32/month + data) and is the #1 surprise bill.** |
| **Security Group (SG)** | Stateful firewall attached to an **instance/ENI**. Allow rules only. |
| **Network ACL (NACL)** | Stateless firewall attached to a **subnet**. Allow and deny rules, evaluated in order. |
| **VPC Endpoint** | Private path to AWS services (S3, DynamoDB, SQS...) without going through the internet or NAT. |

### What makes a subnet "public"?
**Only this:** its route table has a route `0.0.0.0/0 → igw-xxxx`. That's it. (Instances in it
also need a public IP to be reachable.)

A **private** subnet routes `0.0.0.0/0 → nat-xxxx` (or has no internet route at all).

```
Public route table                  Private route table
Destination   Target                Destination   Target
10.0.0.0/16   local                 10.0.0.0/16   local
0.0.0.0/0     igw-0abc              0.0.0.0/0     nat-0def
```
The `local` route is automatic: everything inside a VPC can route to everything else inside it.

### Security Groups vs NACLs

| | Security Group | NACL |
|---|---|---|
| Attached to | Instance / network interface | Subnet |
| Rules | Allow only | Allow **and** Deny, numbered order |
| State | **Stateful**: return traffic is automatically allowed | **Stateless**: you must allow return traffic (ephemeral ports 1024-65535) |
| Default | Deny all in, allow all out | Default NACL allows everything |
| Use it for | Almost everything | Coarse subnet-wide blocks (e.g. block a bad IP range) |

**The SG trick you should use:** a rule's source can be *another security group*.
"DB SG allows 5432 from App SG" means any instance in the App SG can reach the DB, however many
instances there are and whatever their IPs. That's how you chain tiers together.

```
Internet ──443──▶ [ALB SG: allow 443 from 0.0.0.0/0]
                      ──8080──▶ [App SG: allow 8080 from ALB SG]
                                    ──5432──▶ [DB SG: allow 5432 from App SG]
```

### Connecting VPCs and on-prem (know the names)
- **VPC Peering**: connects two VPCs 1:1. Not transitive (A↔B and B↔C does not give you A↔C).
- **Transit Gateway**: a hub-and-spoke router for many VPCs and VPNs. What larger orgs use.
- **Site-to-Site VPN**: encrypted tunnel over the internet to your datacenter.
- **Direct Connect**: a dedicated physical line to AWS.
- **PrivateLink**: expose *one service* privately to other VPCs/accounts (the tech behind interface endpoints).

### DNS & edge
- **Route 53**: AWS's DNS. Hosted zones, record types, plus routing policies (latency-based,
  weighted for canaries, failover with health checks). Its **Alias** records point at AWS
  resources (ALB, CloudFront, S3) and work at the zone apex (`example.com`).
- **CloudFront**: the CDN. It caches content at edge locations close to users and sits in front of S3 or ALBs.

## Debugging checklist: "I can't reach my instance"

1. Does the instance have a **public IP**?
2. Is the subnet's route table sending `0.0.0.0/0` to an **IGW**?
3. Does the **security group** allow the port *from your IP*?
4. Is a **NACL** blocking it (remember: stateless)?
5. Is the process actually **listening** on `0.0.0.0` (not just `127.0.0.1`)? Is there an OS firewall?

**VPC Flow Logs** record accepted/rejected traffic per interface, which shows you *where* it's dropped.

## Lab: build a public subnet by hand (10 min)

Doing this once in the CLI makes every console wizard and Terraform module make sense.

```bash
export AWS_REGION=us-east-1

VPC=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query Vpc.VpcId --output text)
aws ec2 create-tags --resources $VPC --tags Key=Name,Value=lab-vpc
aws ec2 modify-vpc-attribute --vpc-id $VPC --enable-dns-hostnames

SUBNET=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.1.0/24 \
          --availability-zone ${AWS_REGION}a --query Subnet.SubnetId --output text)

# Right now this subnet is PRIVATE: it only has the "local" route. Look:
aws ec2 describe-route-tables --filters Name=vpc-id,Values=$VPC \
  --query 'RouteTables[].Routes[]' --output table

# Make it public: IGW + route + route-table association
IGW=$(aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC

RT=$(aws ec2 create-route-table --vpc-id $VPC --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW
aws ec2 associate-route-table --route-table-id $RT --subnet-id $SUBNET
aws ec2 modify-subnet-attribute --subnet-id $SUBNET --map-public-ip-on-launch   # auto public IPs

# A security group allowing HTTP from anywhere, SSH from only your IP
MYIP=$(curl -s https://checkip.amazonaws.com)
SG=$(aws ec2 create-security-group --group-name lab-web --description "lab web" \
      --vpc-id $VPC --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG --protocol tcp --port 22 --cidr $MYIP/32

aws ec2 describe-route-tables --route-table-ids $RT --query 'RouteTables[].Routes[]' --output table
echo "VPC=$VPC SUBNET=$SUBNET SG=$SG IGW=$IGW RT=$RT"   # keep these for module 03
```

> **Keep this VPC if you're going straight to module 03.** You'll launch an instance into it.
> Otherwise clean up now.

### Cleanup (reverse order; dependencies must go first)
```bash
aws ec2 delete-security-group --group-id $SG
aws ec2 disassociate-route-table --association-id $(aws ec2 describe-route-tables --route-table-ids $RT \
  --query 'RouteTables[0].Associations[0].RouteTableAssociationId' --output text)
aws ec2 delete-route-table --route-table-id $RT
aws ec2 detach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC
aws ec2 delete-internet-gateway --internet-gateway-id $IGW
aws ec2 delete-subnet --subnet-id $SUBNET
aws ec2 delete-vpc --vpc-id $VPC
```

(In real life you'd write this in CloudFormation or Terraform. Module 06 covers that, and
deleting the stack deletes everything in the right order.)

## Check yourself

1. You put a web server in a subnet whose route table has `0.0.0.0/0 → nat-123`. Can users on the internet reach it?
2. Private-subnet instances need to download OS updates. What do you add? And for reading from S3 only?
3. SG allows inbound 443. Do you need an outbound rule for the response?
4. Your NACL allows inbound 443 but outbound only allows port 443. HTTPS requests hang. Why?
5. You need 3 tiers × 2 AZs. How many subnets?

<details><summary>Answers</summary>

1. No. That's a private subnet. NAT only allows outbound-initiated connections. Put the web server (or better, a load balancer) in a public subnet.
2. A NAT Gateway in a public subnet plus a route from the private subnets. For S3 only, a **Gateway VPC Endpoint** for S3 is free and avoids NAT data charges.
3. No. SGs are stateful, so responses are allowed automatically.
4. NACLs are stateless. Responses go back to the client's ephemeral port (1024-65535), which the outbound rule blocks.
5. Six. A subnet lives in exactly one AZ.
</details>

**Next → [03 Compute](03-compute.md)**
