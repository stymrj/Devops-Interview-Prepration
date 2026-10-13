# VPC Hands-On Lab

*8 exercises to make VPC networking muscle memory. Run these in a throwaway Region or your sandbox account — delete the VPC at the end so nothing lingers.*

**Prereqs:** AWS CLI configured (`aws sts get-caller-identity` works), a free-tier account. Nothing here costs more than a few cents if you clean up.

```bash
export AWS_REGION=ap-south-1
aws sts get-caller-identity --query Account --output text
```

---

## Exercise 1: Build a VPC from scratch (no console)

**Goal:** Create a VPC, two subnets, an IGW, and wire the route tables — everything with the CLI.

**Commands:**

```bash
VPC=$(aws ec2 create-vpc --cidr-block 10.99.0.0/16 \
  --query "Vpc.VpcId" --output text)
aws ec2 create-tags --resources $VPC --tags Key=Name,Value=lab-vpc
PUB=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.99.1.0/24 \
  --availability-zone ${AWS_REGION}a --query "Subnet.SubnetId" --output text)
PRIV=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.99.2.0/24 \
  --availability-zone ${AWS_REGION}a --query "Subnet.SubnetId" --output text)
IGW=$(aws ec2 create-internet-gateway --query "InternetGateway.InternetGatewayId" --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC --internet-gateway-id $IGW
RT=$(aws ec2 create-route-table --vpc-id $VPC --query "RouteTable.RouteTableId" --output text)
aws ec2 create-route --route-table-id $RT --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW
aws ec2 associate-route-table --route-table-id $RT --subnet-id $PUB
echo "VPC=$VPC PUB=$PUB PRIV=$PRIV"
```

**Expected output:** `$PUB` gets a route table with `0.0.0.0/0 → igw-xxx` (public); `$PRIV` still sits on the main table (local route only = private).

**Why it matters:** This is the skeleton every production VPC shares — build it by hand once and the console never confuses you again.

---

## Exercise 2: Prove public vs private is just a route

**Goal:** Show that the same subnet flips public/private purely by route-table association.

**Commands:**

```bash
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=$PUB \
  --query "RouteTables[].Routes[].{Dest:DestinationCidrBlock,Gw:GatewayId}" --output table
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=$PRIV \
  --query "RouteTables[].Routes[].DestinationCidrBlock" --output text
aws ec2 disassociate-route-table \
  --association-id $(aws ec2 describe-route-tables --route-table-ids $RT \
  --query "RouteTables[0].Associations[?SubnetId=='$PUB'].RouteTableAssociationId" --output text)
aws ec2 describe-route-tables --route-table-ids $RT --query "RouteTables[0].Associations" --output text
```

**Expected output:** Before disassociation, `$PUB` shows the IGW route and `$PRIV` shows only `local`. After disassociation, the association list is empty — `$PUB` falls back to the main table.

**Why it matters:** It kills the "public subnet" mystique — it's a routing property, not a subnet type. Interviewers love this question.

---

## Exercise 3: Add a NAT gateway and give the private subnet egress

**Goal:** Let private instances reach the internet (outbound only) via a NAT gateway.

**Commands:**

```bash
EIP=$(aws ec2 allocate-address --domain vpc --query "AllocationId" --output text)
NATGW=$(aws ec2 create-nat-gateway --subnet-id $PUB --allocation-id $EIP \
  --query "NatGateway.NatGatewayId" --output text)
aws ec2 wait nat-gateway-available --nat-gateway-ids $NATGW
RTP=$(aws ec2 create-route-table --vpc-id $VPC --query "RouteTable.RouteTableId" --output text)
aws ec2 create-route --route-table-id $RTP --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NATGW
aws ec2 associate-route-table --route-table-id $RTP --subnet-id $PRIV
echo "NATGW=$NATGW"
```

**Expected output:** `nat-gateway-available` wait succeeds; the private route table now shows `0.0.0.0/0 → nat-xxx`.

**Why it matters:** This is the standard pattern for private subnets in every real VPC — app servers that need updates and API calls without inbound exposure.

---

## Exercise 4: Security group referencing (tier-to-tier rules)

**Goal:** Create two security groups where the database group only allows the app group's traffic.

**Commands:**

```bash
SG_APP=$(aws ec2 create-security-group --vpc-id $VPC --group-name lab-app \
  --description "lab app tier" --query GroupId --output text)
SG_DB=$(aws ec2 create-security-group --vpc-id $VPC --group-name lab-db \
  --description "lab db tier" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_DB \
  --protocol tcp --port 5432 --source-group $SG_APP
aws ec2 authorize-security-group-ingress --group-id $SG_APP \
  --protocol tcp --port 8080 --cidr 0.0.0.0/0
aws ec2 describe-security-groups --group-ids $SG_DB \
  --query "SecurityGroups[0].IpPermissions[0].{Port:ToPort,Source:UserIdGroupPairs[0].GroupId}" --output text
```

**Expected output:** The db rule's source is the app security group ID, not a CIDR — IPs can change and the rule still holds.

**Why it matters:** Group-to-group references are how three-tier architectures stay correct as instances churn. This exact pattern comes up in interviews constantly.

---

## Exercise 5: Break a NACL, then feel the difference from security groups

**Goal:** Replace the default NACL with a restrictive one and watch outbound replies get dropped.

**Commands:**

```bash
NACL=$(aws ec2 create-network-acl --vpc-id $VPC --query "NetworkAcl.NetworkAclId" --output text)
ASSOC=$(aws ec2 describe-network-acls --filters Name=vpc-id,Values=$VPC \
  --query "NetworkAcls[?IsDefault==\`true\`].Associations[0].NetworkAclAssociationId" --output text)
aws ec2 replace-network-acl-association --association-id $ASSOC --network-acl-id $NACL
aws ec2 create-network-acl-entry --network-acl-id $NACL --rule-number 100 \
  --protocol 6 --rule-action allow --ingress --cidr-block 10.99.0.0/16 --port-range From=443,To=443
aws ec2 create-network-acl-entry --network-acl-id $NACL --rule-number 100 \
  --protocol 6 --rule-action allow --egress --cidr-block 0.0.0.0/0 --port-range From=1024,To=65535
aws ec2 describe-network-acl-entries --network-acl-id $NACL >/dev/null && echo "nacl ok"
# restore the default NACL when done
DEFNACL=$(aws ec2 describe-network-acls --filters Name=vpc-id,Values=$VPC \
  --query "NetworkAcls[?IsDefault==\`true\`].NetworkAclId" --output text)
aws ec2 replace-network-acl-association --association-id $ASSOC --network-acl-id $DEFNACL
```

**Expected output:** With only 443 ingress and ephemeral-range egress, you see exactly what a stateless filter demands — one missing ephemeral rule and return traffic dies silently.

**Why it matters:** This is the ephemeral-port gotcha from the guide, made real. Once you've broken it, you'll never forget it.

---

## Exercise 6: Reach S3 from a private subnet without NAT (gateway endpoint)

**Goal:** Add a gateway VPC endpoint for S3 and verify the route appears.

**Commands:**

```bash
RTID=$(aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=$PRIV \
  --query "RouteTables[0].RouteTableId" --output text)
aws ec2 create-vpc-endpoint --vpc-id $VPC --service-name com.amazonaws.${AWS_REGION}.s3 \
  --vpc-endpoint-type Gateway --route-table-ids $RTID
aws ec2 describe-route-tables --route-table-ids $RTID \
  --query "RouteTables[0].Routes[?starts_with(GatewayId,'vpce-')].DestinationPrefixListId" --output text
```

**Expected output:** A new route appears with a prefix-list destination (e.g. `pl-xxx`) pointing at the endpoint — S3 traffic now stays off the NAT gateway.

**Why it matters:** Gateway endpoints are free and remove S3/DynamoDB traffic from your NAT bill — a classic cost-optimization move interviewers ask about.

---

## Exercise 7: Turn on VPC Flow Logs and read a REJECT

**Goal:** Capture rejected traffic and find the dropping layer.

**Commands:**

```bash
aws ec2 create-flow-logs --resource-type VPC --resource-ids $VPC \
  --traffic-type REJECT --log-destination-type cloud-watch-logs \
  --log-group-name lab-vpc-flowlogs --deliver-logs-permission-arn REPLACE_WITH_FLOWLOG_ROLE_ARN
sleep 60
aws logs filter-log-events --log-group-name lab-vpc-flowlogs \
  --limit 5 --query "events[].message" --output text | head -5
```

**Expected output:** Flow log records showing source/dest IPs, ports, and `REJECT` actions — the exact evidence you cite when debugging connectivity.

**Why it matters:** "Check Flow Logs" is the senior answer to every VPC debugging question. Doing it once makes it second nature.

> **Note:** This needs an IAM role with a trust policy for `vpc-flow-logs.amazonaws.com` — create it in the console once, then paste the ARN above. Delete the log group afterward to avoid charges.

---

## Exercise 8: Tear it all down cleanly

**Goal:** Delete everything in dependency order so the lab account is pristine.

**Commands:**

```bash
aws ec2 delete-nat-gateway --nat-gateway-id $NATGW
aws ec2 delete-vpc-endpoint --vpc-endpoints $(aws ec2 describe-vpc-endpoints \
  --filters Name=vpc-id,Values=$VPC --query "VpcEndpoints[0].VpcEndpointId" --output text)
sleep 45
aws ec2 release-address --allocation-id $EIP
aws ec2 delete-security-group --group-id $SG_DB
aws ec2 delete-security-group --group-id $SG_APP
aws ec2 delete-subnet --subnet-id $PUB
aws ec2 delete-subnet --subnet-id $PRIV
aws ec2 delete-route-table --route-table-id $RT
aws ec2 delete-route-table --route-table-id $RTP
aws ec2 detach-internet-gateway --vpc-id $VPC --internet-gateway-id $IGW
aws ec2 delete-internet-gateway --internet-gateway-id $IGW
aws ec2 delete-network-acl --network-acl-id $NACL
aws ec2 delete-vpc --vpc-id $VPC
aws ec2 describe-vpcs --vpc-ids $VPC 2>&1 | grep -q InvalidVpcID.NotFound && echo "vpc gone"
```

**Expected output:** `vpc gone` — nothing left. Verify in the console that no NAT gateway or EIP is still billing.

**Why it matters:** NAT gateways and EIPs bill by the hour whether you use them or not. Clean teardown is a habit, not an afterthought.

---

**Cleanup checklist:** NAT gateway deleted · EIP released · VPC deleted · Flow Logs log group deleted. If any step above failed, delete leftovers from the VPC console before logging off.
