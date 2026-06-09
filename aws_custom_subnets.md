# SOP: Creating Custom Subnets and Managing CIDR Overlap

## 📝 1. OBJECTIVE (The "Why")
**Objective:** To carve out a specifically sized network block (Subnet) from a larger Virtual Private Cloud (VPC) CIDR without causing IP address conflicts.
**Context:** When provisioning infrastructure in AWS, servers must be placed in subnets. If using the AWS Default VPC (typically `172.31.0.0/16`), AWS pre-allocates several `/20` subnets. Attempting to create a standard `/24` subnet starting at `172.31.0.0` will result in an overlap collision. This procedure ensures you find available "white space" in the network and provision the subnet successfully.

**Real-World Example:** Segregating a VPC into a "Public Subnet" for web servers that face the internet, and a "Private Subnet" for databases that should never be directly accessible from the outside.

---

## ⚙️ 2. TARGET INFRASTRUCTURE / PREREQUISITES (The "Where")
- **Platform:** AWS Management Console or AWS CLI
- **Target System:** AWS VPC (e.g., `172.31.0.0/16`)
- **Required Permissions:** IAM user/role with `AmazonVPCFullAccess` or specific `ec2:CreateSubnet` and `ec2:DescribeSubnets` permissions.
- **Prerequisites:** You must know your target VPC ID (e.g., `vpc-0123456789abcdef0`).

---

## 🛠️ 3. EXECUTION (The "How")

### Step 1: Audit Existing Network Fences (Avoid Overlap)
Before creating a subnet, you must find empty IP space.

**Command (CLI):**
```bash
aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=vpc-YOUR_VPC_ID" \
    --query "Subnets[*].[SubnetId,CidrBlock]" \
    --output table
```

**What it does:** Lists all currently carved-out subnets in your VPC.
**Why:** You must identify the highest IP range currently in use (e.g., AWS defaults usually consume up to `172.31.47.255`).

### Step 2: Calculate Your Target CIDR Block
Select a `/24` block (256 IPs) that sits above the existing subnets.

- **Example VPC:** `172.31.0.0/16`
- **Safe Range:** Choose a high third octet. E.g., `172.31.100.0/24`.

**Common Mistake:** Forgetting that AWS reserves 5 IP addresses per subnet. A `/24` provides 256 total IPs, but only 251 are usable for your EC2 instances. The first four and the last one are reserved by AWS for routing, DNS, and broadcast functions.

### Step 3: Create the Subnet

#### Option A: Execution via AWS CLI (Automation Standard)
```bash
aws ec2 create-subnet \
    --vpc-id vpc-YOUR_VPC_ID \
    --cidr-block 172.31.100.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-1}]'
```

**What it does:** Issues an API call to AWS to partition the `172.31.100.0` to `172.31.100.255` block into a named subnet in a specific data center (Availability Zone).

#### Option B: Execution via AWS Console (Visual Method)
1. Navigate to **VPC Console -> Subnets -> Click Create subnet**.
2. Select the target **VPC ID** from the dropdown.
3. **Subnet name:** `public-subnet-1`
4. **Availability Zone:** Select any available zone (e.g., `us-east-1a`).
5. **IPv4 CIDR block:** Enter `172.31.100.0/24`.
6. Click **Create subnet**.

---

## ✅ 4. VERIFICATION & TROUBLESHOOTING (The "Proof")

### Verification:
**Verify State:** Ensure the subnet state is `Available`.
```bash
aws ec2 describe-subnets --subnet-ids subnet-YOUR_NEW_SUBNET_ID --query "Subnets[*].State"
```

**Verify Capacity:** Check the "Available IPv4 addresses" column in the console. It should read exactly 251.

### Troubleshooting:

| Error | Meaning | Solution |
| :--- | :--- | :--- |
| **CIDR Address overlaps with existing Subnet** | You are trying to build a subnet on IPs that are already claimed by another subnet. | Run Step 1 to map existing CIDRs. Pick a higher third octet (e.g., jump from `.0.0/24` to `.100.0/24`). |
| **The CIDR block is out of range** | The subnet you are trying to create is outside the boundaries of the parent VPC. | Ensure your subnet starts with the same locked bits as the VPC (e.g., if VPC is `172.31.0.0/16`, subnet MUST start with `172.31.X.X`). |
| **VpcIdNotFound** | The VPC ID provided does not exist in the current AWS Region. | Verify you are operating in the correct region (e.g., `us-east-1` vs `eu-west-1`) and check the VPC ID string. |
