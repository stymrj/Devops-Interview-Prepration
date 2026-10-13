# VPC (Subnets, Route Tables, NAT, Security Groups) Interview Preparation Guide

*How to Answer VPC Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [VPCs and Subnets](#vpcs-and-subnets)
2. [CIDR and IP Addressing](#cidr-and-ip-addressing)
3. [Route Tables and Routing](#route-tables-and-routing)
4. [Internet Gateways and NAT](#internet-gateways-and-nat)
5. [Security Groups and NACLs](#security-groups-and-nacls)
6. [VPC Endpoints and Peering](#vpc-endpoints-and-peering)
7. [Interview Traps and Real Scenarios](#interview-traps-and-real-scenarios)

---

## VPCs and Subnets

### Q1: What is a VPC? Why not just deploy in the default VPC?

**How to Answer:**

"A VPC is my own isolated network inside AWS — my own IP range, subnets, and routing, logically sealed off from everyone else's. The default VPC exists so beginners can launch an instance in one click, but it comes with public subnets everywhere and one flat security posture. I always build a custom VPC because I need private subnets for databases and internal services, plus separate route tables per tier. For anything production, default VPC is just a shortcut I delete."

**Key Point:** "The default VPC is a quickstart, not an architecture — production gets a custom VPC with real network tiers."

---

### Q2: What makes a subnet public vs private?

**How to Answer:**

"There's nothing magical about the name — a subnet is public only if its route table sends `0.0.0.0/0` traffic to an Internet Gateway. Same subnet, remove that route, and it's private. My pattern is public subnets for load balancers and NAT gateways only, private subnets for app servers and data, and isolated subnets for databases with no outbound route at all. I spread each tier across three AZs, because a subnet lives in exactly one AZ and a single-AZ tier is a single point of failure."

```bash
aws ec2 create-subnet --vpc-id vpc-0abc123 --cidr-block 10.0.1.0/24 \
  --availability-zone ap-south-1a
```

**Key Point:** "Public means the route table points at an IGW — the subnet label itself decides nothing."

---

## CIDR and IP Addressing

### Q3: How do you size a VPC CIDR block? What are the common interview traps here?

**How to Answer:**

"I pick a `/16` for any VPC I expect to grow — 65,536 addresses sounds huge until AZs, tiers, and per-subnet waste eat into it. AWS reserves 5 IPs in every subnet (the network, VPC router, DNS, future use, and broadcast), so a `/24` gives me 251 usable addresses, not 256. The trap interviewers love: two VPCs with overlapping CIDR can't be peered, so I plan ranges up front from a private block like `10.0.0.0/8` and keep a small registry. IPv4 exhaustion is real inside big orgs, so I also enable dual-stack IPv6 on newer VPCs where it makes sense."

**Key Point:** "Plan the CIDR before you build — overlapping ranges can't be peered, and every subnet loses 5 IPs."

---

### Q4: How many IPs do you actually get in a /24 subnet, and why?

**How to Answer:**

"You get 251 usable IPs out of 256 in a `/24`. AWS keeps five: the network address, the VPC router at `x.x.x.1`, the DNS resolver at `x.x.x.2`, one reserved for future use, and the broadcast address. It trips people up because the numbers feel arbitrary until you hit an 'insufficient addresses' error on a big EKS cluster. I always count the usable number, not the CIDR total, when sizing subnets for node pools."

**Key Point:** "A /24 gives 251 usable IPs — AWS reserves five per subnet, always."

---

## Route Tables and Routing

### Q5: How do route tables work? What is the local route?

**How to Answer:**

"Every subnet is attached to exactly one route table, and the route table decides where packets go based on the destination CIDR. There's always an implicit `local` route for the whole VPC CIDR — it can't be deleted or overridden, and it's why resources inside the VPC can always talk to each other. I add specific routes on top: `0.0.0.0/0` to the IGW for public subnets, `0.0.0.0/0` to the NAT gateway for private ones, and prefix routes for peering. Longest-prefix-match wins, so a specific `10.1.0.0/16` route beats the default route every time."

**Key Point:** "The local route is always there and always wins inside the VPC — everything else is longest-prefix-match."

---

### Q6: What is the difference between a route table and the main route table?

**How to Answer:**

"The main route table is the default — any subnet I don't explicitly associate falls back to it, which is why I treat it as a trap for accidental exposure. My rule is to never put a `0.0.0.0/0 → IGW` route on the main table; if someone creates a subnet and forgets the association, it lands on main and inherits whatever I left there. I lock the main table down with just the local route and associate every subnet explicitly. That way nothing is public by accident."

**Key Point:** "Keep the main route table boring — unassociated subnets inherit it, so never leave a public route there."

---

## Internet Gateways and NAT

### Q7: How does traffic reach the internet from a public subnet?

**How to Answer:**

"Three things have to line up: the instance needs a public IP, the subnet's route table sends `0.0.0.0/0` to an Internet Gateway, and the security group allows the traffic. The IGW itself does 1:1 NAT between the public IP and the private IP — it's horizontally scaled and fully managed, I never maintain it. Most debugging sessions start here because exactly one of those three is always the broken piece. I check the route table first, since that's where the mistake usually is."

**Key Point:** "Public internet access = public IP + 0.0.0.0/0 → IGW + open security group — miss one and nothing works."

---

### Q8: How do private instances reach the internet for updates and API calls?

**How to Answer:**

"They go through a NAT Gateway sitting in a public subnet — the private subnet's route table sends `0.0.0.0/0` to the NAT gateway, and it forwards outbound traffic through its own Elastic IP. NAT gateways are stateful one-way doors: outbound connections work, inbound can't be initiated from the internet. The classic gotcha is putting one NAT gateway per VPC — it's zonal, so if that AZ goes down, every private subnet loses egress. I deploy one NAT gateway per AZ for anything that matters, and I watch the data-processing charges since NAT isn't free."

```bash
aws ec2 create-nat-gateway --subnet-id subnet-0pub123 \
  --allocation-id eipalloc-0def456
```

**Key Point:** "NAT gateway gives private subnets one-way outbound access — but deploy it per AZ or you have a zonal single point of failure."

---

### Q9: When would you use a NAT instance instead of a NAT gateway?

**How to Answer:**

"Almost never in production. A NAT instance is a self-managed EC2 box doing the same job, so I'm on the hook for patching, HA, and scaling it myself. The only real cases are extreme cost sensitivity in a lab, or when I need custom traffic inspection a gateway can't do. For everything real, the managed NAT gateway wins — it's redundant inside the AZ and I pay per hour plus data rather than babysitting an instance. If an interviewer pushes, I mention bastion-style traffic logging as the one honest NAT-instance use case I've seen."

**Key Point:** "NAT instances are DIY and fragile — the managed gateway wins unless you need custom traffic inspection."

---

## Security Groups and NACLs

### Q10: What's the difference between a security group and a NACL?

**How to Answer:**

"Security groups are stateful firewalls on the instance's ENI — if I allow inbound on port 443, the return traffic is automatically allowed. NACLs are stateless subnet-level filters — I have to allow the ephemeral return ports explicitly or responses get dropped. Security groups only support allow rules; NACLs support explicit denies too, which is their one superpower. I use security groups for everything day to day and NACLs as a coarse backstop, like blocking a known-bad IP range across a whole subnet."

**Key Point:** "Security groups are stateful and allow-only; NACLs are stateless and can deny — that's the whole interview answer."

---

### Q11: What's the ephemeral port gotcha with NACLs?

**How to Answer:**

"When my instance makes an outbound request, the reply comes back on a random ephemeral port, usually 1024-65535. With a stateless NACL I have to explicitly allow inbound traffic on those ports or every response gets dropped — meanwhile the security group would have handled it automatically because it's stateful. It's the number-one reason people break SSH or HTTPS right after 'securing' a NACL. My default NACL rules are: allow all outbound, allow inbound ephemeral ports plus the specific service ports, and a deny-all at the end. Custom NACL rules are evaluated in number order, lowest first."

**Key Point:** "Stateless NACLs must allow inbound ephemeral ports for return traffic — forget that and you silently break everything."

---

## VPC Endpoints and Peering

### Q12: How do private subnets reach AWS services like S3 without going to the internet?

**How to Answer:**

"With VPC endpoints. A gateway endpoint for S3 and DynamoDB is free and routes through a prefix-list entry in the route table — no NAT gateway needed, no data charges. An interface endpoint spins up ENIs with private IPs in my subnets for everything else — SSM, ECR, Secrets Manager — and talks to the service over AWS's private backbone. The gotcha is endpoint policies: an overly tight policy silently denies S3 access and the error looks like a network problem. When my private instances talk to AWS services, endpoints are always the answer instead of burning NAT gateway money."

**Key Point:** "Endpoints keep AWS-service traffic off the internet — gateway endpoints are free, interface endpoints cost per hour and per GB."

---

### Q13: Compare VPC peering, Transit Gateway, and PrivateLink.

**How to Answer:**

"Peering is a direct 1:1 link between two VPCs — no transitive routing, so three VPCs need three peerings and it gets messy fast. Transit Gateway is a hub that all VPCs attach to with one connection each, and it handles routing between them — my default for ten-plus VPCs. PrivateLink exposes a single service privately without connecting the networks at all, which is what I use when a partner or another team needs just my API, not my VPC. The peering trap interviewers love: overlapping CIDRs can't be peered, full stop."

**Key Point:** "Peering for a couple of VPCs, Transit Gateway for many, PrivateLink when you only want to share a service — never peer overlapping CIDRs."

---

## Interview Traps and Real Scenarios

### Q14: Walk me through debugging: an EC2 instance in a private subnet can't reach the internet. What do you check?

**How to Answer:**

"I start at the route table — does the subnet have `0.0.0.0/0` pointing at a NAT gateway? Then I check the NAT gateway itself: is it in a public subnet, does it have an Elastic IP, and is the public subnet's route table pointed at an IGW? Next I verify the security group allows outbound and the NACL isn't blocking ephemeral return ports. Then I check VPC Flow Logs for REJECT entries to see exactly which layer dropped the traffic. Nine times out of ten it's the route table or the NAT gateway sitting in the wrong subnet."

```bash
aws ec2 describe-flow-logs --filter Name=vpc-id,Values=vpc-0abc123
```

**Key Point:** "Debug top-down: route table, NAT gateway placement, security groups, NACLs — Flow Logs tell you which layer dropped it."

---

### Q15: How do you secure a three-tier VPC (web, app, database)?

**How to Answer:**

"Three subnet tiers across three AZs. Web tier in public subnets behind an ALB, app tier in private subnets that only accept traffic from the web security group, and the database in isolated subnets that only accept 5432 from the app security group. Security groups reference each other by ID instead of CIDR blocks — that way the rules stay correct when IPs change. NACLs stay at the default allow-all because I don't want to fight stateless rules on top of a clean security-group design. The database gets no internet route at all, not even via NAT — patching happens through VPC endpoints."

**Key Point:** "Security groups referencing each other by ID, database in isolated subnets with no internet route at all."
