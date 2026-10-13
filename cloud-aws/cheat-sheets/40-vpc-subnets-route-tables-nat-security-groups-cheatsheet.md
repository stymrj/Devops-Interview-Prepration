# VPC Cheat Sheet

*Dense command reference for VPC networking — subnets, routing, NAT, firewalls, endpoints.*

## VPCs

```bash
# Create / inspect
aws ec2 create-vpc --cidr-block 10.0.0.0/16
aws ec2 describe-vpcs --vpc-ids vpc-xxx --query "Vpcs[0].{Cidr:CidrBlock,State:State}"
aws ec2 modify-vpc-attribute --vpc-id vpc-xxx --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id vpc-xxx --enable-dns-support
aws ec2 delete-vpc --vpc-id vpc-xxx            # fails if anything still attached
```

## Subnets

```bash
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.1.0/24 --availability-zone ap-south-1a
aws ec2 modify-subnet-attribute --subnet-id subnet-xxx --map-public-ip-on-launch
aws ec2 describe-subnets --filters Name=vpc-id,Values=vpc-xxx \
  --query "Subnets[].{Id:SubnetId,Cidr:CidrBlock,AZ:AvailabilityZone,Avail:AvailableIpAddressCount}" --output table
```

- Subnet lives in exactly one AZ. A `/24` = 251 usable IPs (AWS reserves 5).
- Public vs private is decided by the route table, not the subnet.

## Route tables

```bash
aws ec2 create-route-table --vpc-id vpc-xxx
aws ec2 associate-route-table --route-table-id rtb-xxx --subnet-id subnet-xxx
aws ec2 create-route --route-table-id rtb-xxx --destination-cidr-block 0.0.0.0/0 --gateway-id igw-xxx
aws ec2 create-route --route-table-id rtb-xxx --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-xxx
aws ec2 create-route --route-table-id rtb-xxx --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id pcx-xxx
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=subnet-xxx \
  --query "RouteTables[0].Routes" --output table
```

- `local` route (whole VPC CIDR) is implicit and always wins inside the VPC.
- Longest-prefix-match decides between specific and default routes.
- Keep the main route table free of public routes — unassociated subnets inherit it.

## Internet gateways

```bash
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --vpc-id vpc-xxx --internet-gateway-id igw-xxx
aws ec2 detach-internet-gateway --vpc-id vpc-xxx --internet-gateway-id igw-xxx
aws ec2 delete-internet-gateway --internet-gateway-id igw-xxx
```

- One IGW per VPC. Public access = public IP + `0.0.0.0/0 → IGW` + open SG.

## NAT gateways

```bash
aws ec2 allocate-address --domain vpc                       # get EIP first
aws ec2 create-nat-gateway --subnet-id subnet-public --allocation-id eipalloc-xxx
aws ec2 wait nat-gateway-available --nat-gateway-ids nat-xxx
aws ec2 describe-nat-gateways --filter Name=vpc-id,Values=vpc-xxx \
  --query "NatGateways[].{Id:NatGatewayId,State:State,Subnet:SubnetId}" --output table
aws ec2 delete-nat-gateway --nat-gateway-id nat-xxx
```

- NAT gateway must sit in a **public** subnet. Zonal — deploy one per AZ.
- One-way only: outbound allowed, inbound can't be initiated. Billed per hour + data.

## Security groups

```bash
aws ec2 create-security-group --vpc-id vpc-xxx --group-name web --description "web tier"
aws ec2 authorize-security-group-ingress --group-id sg-xxx --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id sg-db --protocol tcp --port 5432 --source-group sg-app
aws ec2 revoke-security-group-ingress --group-id sg-xxx --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 describe-security-groups --group-ids sg-xxx --query "SecurityGroups[0].IpPermissions"
```

- Stateful (return traffic auto-allowed). Allow rules only. Attached to ENIs, not subnets.
- Reference groups by ID (`--source-group`) instead of CIDR for tier-to-tier rules.

## NACLs

```bash
aws ec2 create-network-acl --vpc-id vpc-xxx
aws ec2 create-network-acl-entry --network-acl-id acl-xxx --rule-number 100 \
  --protocol 6 --rule-action allow --ingress --cidr-block 0.0.0.0/0 --port-range From=443,To=443
aws ec2 create-network-acl-entry --network-acl-id acl-xxx --rule-number 100 \
  --protocol 6 --rule-action allow --egress --cidr-block 0.0.0.0/0 --port-range From=1024,To=65535
aws ec2 replace-network-acl-association --association-id aclassoc-xxx --network-acl-id acl-xxx
```

- Stateless — must allow ephemeral return ports (1024-65535) explicitly.
- Rules evaluated lowest number first; supports explicit deny.

## VPC endpoints

```bash
# Gateway endpoint (S3, DynamoDB — free)
aws ec2 create-vpc-endpoint --vpc-id vpc-xxx --service-name com.amazonaws.ap-south-1.s3 \
  --vpc-endpoint-type Gateway --route-table-ids rtb-xxx
# Interface endpoint (SSM, ECR, Secrets Manager — paid)
aws ec2 create-vpc-endpoint --vpc-id vpc-xxx --service-name com.amazonaws.ap-south-1.ssm \
  --vpc-endpoint-type Interface --subnet-ids subnet-xxx --security-group-ids sg-xxx
```

## Peering & Transit Gateway

```bash
aws ec2 create-vpc-peering-connection --vpc-id vpc-aaa --peer-vpc-id vpc-bbb
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-xxx
# then add routes both ways (peer CIDR → pcx-xxx)
```

- Peering: non-transitive, no overlapping CIDRs, no edge-to-edge routing.
- Transit Gateway: hub for many VPCs, one attachment each, central routing.

## Flow Logs (debugging)

```bash
aws ec2 create-flow-logs --resource-type VPC --resource-ids vpc-xxx \
  --traffic-type REJECT --log-destination-type cloud-watch-logs \
  --log-group-name vpc-flowlogs --deliver-logs-permission-arn arn:aws:iam::123456789012:role/flowlog-role
aws logs filter-log-events --log-group-name vpc-flowlogs --limit 10 \
  --query "events[].message" --output text
```

## One-line interview answers

| Question | Answer |
|---|---|
| Public vs private subnet? | Route table: `0.0.0.0/0 → IGW` = public. Nothing else decides. |
| Usable IPs in /24? | 251 — AWS reserves 5 per subnet. |
| SG vs NACL? | SGs stateful, allow-only, on ENIs. NACLs stateless, can deny, on subnets. |
| Private subnet internet? | `0.0.0.0/0 → NAT gateway` (in a public subnet). One per AZ. |
| NACL ephemeral gotcha? | Stateless — must allow inbound 1024-65535 for return traffic. |
| S3 from private subnet, no NAT? | Gateway VPC endpoint — free, adds a prefix-list route. |
| Peering limits? | Non-transitive, no overlapping CIDRs. |
| Main route table rule? | Keep it minimal — unassociated subnets inherit it. |
