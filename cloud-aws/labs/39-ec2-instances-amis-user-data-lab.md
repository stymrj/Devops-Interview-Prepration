# EC2 Hands-On Lab

*8 exercises to get EC2 into your muscle memory. Work on a free-tier-eligible setup so this costs nothing.*

**Prereqs:** AWS CLI configured (`aws sts get-caller-identity` works), a default VPC, SSH key pair in your Region. Set `KEY=my-key` and `SG` once.

```bash
export KEY=my-key
aws ec2 describe-key-pairs --key-names $KEY >/dev/null && echo "key ok"
```

---

## Exercise 1: Launch your first instance and connect

**Goal:** Launch an Amazon Linux instance and SSH into it.

**Commands:**

```bash
AMI=$(aws ec2 describe-images --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
  --query "sort_by(Images,&CreationDate)[-1].ImageId" --output text)
SG=$(aws ec2 create-security-group --group-name lab-ssh \
  --description "lab ssh" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG \
  --protocol tcp --port 22 --cidr $(curl -s ifconfig.me)/32
ID=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro \
  --key-name $KEY --security-group-ids $SG \
  --query "Instances[0].InstanceId" --output text)
aws ec2 wait instance-running --instance-ids $ID
IP=$(aws ec2 describe-instances --instance-ids $ID \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
ssh -i ~/.ssh/$KEY.pem ec2-user@$IP
```

**Expected output:** SSH drops you into the Amazon Linux shell.

**Why it matters:** This is the atomic unit of EC2 — everything else is variations on this launch.

---

## Exercise 2: Watch a user-data script bootstrap a web server

**Goal:** Launch an instance that configures itself via user data.

**Commands:**

```bash
cat > user-data.sh <<'EOF'
#!/bin/bash -xe
yum update -y
yum install -y nginx
systemctl enable --now nginx
echo "user-data works" > /usr/share/nginx/html/index.html
EOF
ID2=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro \
  --key-name $KEY --security-group-ids $SG \
  --user-data file://user-data.sh \
  --query "Instances[0].InstanceId" --output text)
```

Then allow HTTP on the security group and check the logs:

```bash
aws ec2 authorize-security-group-ingress --group-id $SG \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 wait instance-running --instance-ids $ID2
IP2=$(aws ec2 describe-instances --instance-ids $ID2 \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
curl http://$IP2
```

**Expected output:** `user-data works`. On the instance: `tail /var/log/cloud-init-output.log` shows the script running.

**Why it matters:** User data is last-mile customization — seeing it work end-to-end kills half of all EC2 confusion.

---

## Exercise 3: Build your own golden AMI

**Goal:** Customize an instance, bake it into an AMI, launch a clone.

**Commands:**

```bash
aws ec2 create-image --instance-id $ID2 \
  --name "lab-golden-$(date +%F)" --no-reboot \
  --query ImageId --output text
aws ec2 wait image-available --image-ids ami-NEWID  # use the printed id
```

Then launch from your AMI and curl it — nginx and the page should be there with zero user data.

**Expected output:** The clone serves `user-data works` immediately after boot.

**Why it matters:** This is the golden-image workflow from the guide — bake once, launch identically forever.

---

## Exercise 4: Compare pricing — find today's Spot price

**Goal:** See the real discount on Spot for your instance type.

**Commands:**

```bash
aws ec2 describe-spot-price-history --instance-types t3.micro \
  --product-descriptions "Linux/UNIX" \
  --start-time $(date -u +%FT%TZ) \
  --query "SpotPriceHistory[0].{AZ:AvailabilityZone,Price:SpotPrice}"
```

**Expected output:** A price like `0.0034` — a fraction of the On-Demand rate.

**Why it matters:** Pricing questions are easy marks in interviews once you've actually looked at the numbers.

---

## Exercise 5: Snapshot and restore an EBS volume

**Goal:** Back up a volume and prove you can get the data back.

**Commands:**

```bash
VOL=$(aws ec2 describe-instances --instance-ids $ID \
  --query "Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId" \
  --output text)
SNAP=$(aws ec2 create-snapshot --volume-id $VOL \
  --description "lab-snapshot" --query SnapshotId --output text)
aws ec2 wait snapshot-completed --snapshot-ids $SNAP
aws ec2 create-volume --snapshot-id $SNAP --availability-zone ap-south-1a \
  --query VolumeId --output text
```

**Expected output:** A new volume created from the snapshot, ready to attach.

**Why it matters:** Snapshot/restore is the core of EC2 backup strategy — interviewers expect you to have done it.

---

## Exercise 6: Break and fix — burstable credit exhaustion

**Goal:** Observe CPU credit drain on a T instance.

**Commands:**

```bash
# Burn CPU on the instance for a few minutes
ssh -i ~/.ssh/$KEY.pem ec2-user@$IP \
  "cat /dev/zero > /dev/null &" 
sleep 300
aws cloudwatch get-metric-statistics --namespace AWS/EC2 \
  --metric-name CPUCreditBalance \
  --dimensions Name=InstanceId,Value=$ID \
  --start-time $(date -u -d '30 min ago' +%FT%TZ) \
  --end-time $(date -u +%FT%TZ) --period 300 --statistics Average \
  --query "Datapoints[*].Average"
```

**Expected output:** The balance drops toward zero. Stop the burn job afterwards (`kill %1` on the instance).

**Why it matters:** You can't explain the T-family throttling trap convincingly until you've watched the credit line fall.

---

## Exercise 7: Lock down the security group

**Goal:** Practice least-privilege rules and verify they're stateful.

**Commands:**

```bash
# Remove the 0.0.0.0/0 HTTP rule, keep only your IP
aws ec2 revoke-security-group-ingress --group-id $SG \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG \
  --protocol tcp --port 80 --cidr $(curl -s ifconfig.me)/32
aws ec2 describe-security-groups --group-ids $SG \
  --query "SecurityGroups[0].IpPermissions"
```

**Expected output:** HTTP allowed only from your IP; SSH from Exercise 1 still works both ways.

**Why it matters:** Security-group hygiene is a daily DevOps task — and `0.0.0.0/0` on SSH is an interview red flag.

---

## Exercise 8: Clean up — terminate everything and verify zero cost

**Goal:** Destroy all lab resources and confirm nothing is running.

**Commands:**

```bash
aws ec2 terminate-instances --instance-ids $ID $ID2
aws ec2 wait instance-terminated --instance-ids $ID $ID2
aws ec2 delete-security-group --group-id $SG
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].InstanceId"
aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query "Volumes[].VolumeId"
```

**Expected output:** Empty lists — no running instances, no orphaned volumes.

**Why it matters:** The cheapest EC2 skill is cleanup. Orphaned volumes are the classic surprise bill.

---

*Done? You can now launch, bootstrap, image, price, back up, and clean up EC2 — that's production-level fluency for the interview room.*
