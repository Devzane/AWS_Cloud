# SOP: AWS Key Pair Provisioning & Cloud Authentication

## 📝 Objective

To establish secure, passwordless access to AWS Elastic Compute Cloud (EC2) instances. Cloud environments enforce Asymmetric Cryptography to prevent brute-force attacks. This operation provisions a public/private key pair via Infrastructure as Code (IaC) tools, securely storing the public key in the AWS database while isolating the private key on the local engineering machine.

> **Note:** In SRE and Cloud Architecture, treating `.pem` files as highly sensitive secrets is critical. Compromise of a private key file grants root-level access to the associated infrastructure.

---

## ⚙️ Target Infrastructure

- **Cloud Provider:** Amazon Web Services (AWS)
- **Service:** Elastic Compute Cloud (EC2)
- **Core Tool:** AWS CLI (`aws ec2`)
- **Cryptography:** RSA (Rivest–Shamir–Adleman) or ED25519

---

## 🏗️ The Architectural Framework

When you provision a Key Pair in AWS, two linked cryptographic files are handled differently:

1. **The Public Key (The Lock):** AWS keeps this. When an EC2 instance boots up for the first time, a hidden cloud service called `cloud-init` automatically injects this public key text directly into the virtual server's `/home/<user>/.ssh/authorized_keys` file.
2. **The Private Key (The Physical Key):** AWS hands this to you exactly once via standard output. AWS does not save a copy of this key. If you lose it, AWS cannot recover it.

---

## 🛠️ Core Execution Scenarios

### Scenario A: Generating a New Key Pair via CLI (Standard)

We use the AWS API to generate the cryptographic math. We must use specific queries to extract only the raw private key text and redirect it into a local `.pem` file.

```bash
aws ec2 create-key-pair --key-name datacenter-kp --key-type rsa --query 'KeyMaterial' --output text > datacenter-kp.pem
```

**Flag Breakdown:**
- `--key-name`: The logical database name AWS will use to store the Public Key lock.
- `--key-type`: Specifies the cryptographic algorithm (`rsa` is older/standard, `ed25519` is modern/faster).
- `--query 'KeyMaterial'`: Filters the massive JSON response to output only the raw string of the private key.
- `--output text`: Strips all JSON formatting quotes and brackets.
- `> datacenter-kp.pem`: Redirects the standard output into a physical file on the local machine.

---

### Scenario B: Enforcing Local Security (The "400 Rule")

Linux SSH clients possess a built-in security failsafe: if a private key file is readable by other users on the local machine (e.g., `644` permissions), the SSH daemon assumes the key is compromised and the connection is instantly aborted.

You must lock the file down to owner-read-only immediately:

```bash
chmod 400 datacenter-kp.pem
```

---

## 🚀 Advanced Cloud Operations

### 1. Importing an Existing Local Key (Pro SRE Move)

Senior engineers rarely let AWS generate their keys. Instead, they generate the key locally on their laptop using `ssh-keygen`, and upload the public half to AWS. This guarantees the private key never traveled over the internet.

```bash
# 1. Generate key locally
ssh-keygen -t ed25519 -f ~/.ssh/my_custom_key

# 2. Upload the public half to AWS
aws ec2 import-key-pair --key-name imported-kp --public-key-material fileb://~/.ssh/my_custom_key.pub
```

### 2. Deleting a Key Pair

When rotating credentials or cleaning up old infrastructure, you must wipe the public lock from the AWS database.

```bash
aws ec2 delete-key-pair --key-name datacenter-kp
```

---

## 📋 Quick Reference: Command Matrix

| Operation | Command | Description |
|-----------|---------|-------------|
| **Create Key** | `aws ec2 create-key-pair ...` | Generates a new key pair on AWS |
| **Delete Key** | `aws ec2 delete-key-pair --key-name NAME` | Removes public key from AWS database |
| **List Keys** | `aws ec2 describe-key-pairs` | Shows all registered public keys |
| **Import Key** | `aws ec2 import-key-pair ...` | Uploads a locally generated public key |
| **Secure Key** | `chmod 400 key.pem` | Locks down local private key permissions |

---

## ✅ Verification & Auditing

Do not assume the AWS API accepted the command. Query the AWS database to mathematically prove the Public Key was registered.

```bash
aws ec2 describe-key-pairs --key-names datacenter-kp
```

**Expected Output:** A JSON block containing the `KeyPairId` and cryptographic `KeyFingerprint`.

---

## 🔐 Security Best Practices

- **The `.gitignore` Rule:** NEVER commit a `.pem` file to a Git repository. Hackers use bots to scan public GitHub repos for `.pem` files, and they will hijack your AWS account in seconds to mine cryptocurrency.
- **Key Rotation:** For production environments, key pairs should be rotated every 90-180 days. Generate a new key, inject the new public key into the servers, verify access, and delete the old key.
- **IAM Roles over Keys:** For machine-to-machine authentication (e.g., an app talking to a database), always use IAM Roles attached to the EC2 instance instead of embedding static `.pem` keys or passwords in code.

---

## 🐛 Troubleshooting & Edge Cases

### Problem 1: `An error occurred (InvalidKeyPair.Duplicate)... The keypair already exists`

- **Root Cause:** AWS already has a public key registered under this name. AWS does not save your private key, so if you deleted your local `.pem` file by accident, you cannot just run the create command again to get it back.
- **Resolution:** You must explicitly delete the orphaned lock from the AWS database before generating a new one using the exact same name:
  ```bash
  aws ec2 delete-key-pair --key-name datacenter-kp
  ```

### Problem 2: `An error occurred (InvalidKeyPair.NotFound)... The key pair 'datacenter-kp.pem' does not exist`

- **Root Cause:** This occurs during the verification (`describe`) or deletion step if you append the `.pem` extension to the key name.
- **Resolution:** AWS does not know about the physical `.pem` file on your local hard drive. It only knows the logical database name. Omit `.pem` and query using strictly the `--key-name` (e.g., `datacenter-kp`).

### Problem 3: `WARNING: UNPROTECTED PRIVATE KEY FILE!`

- **Root Cause:** You skipped the security lockdown step. The local Linux permissions are too broad (e.g., `644` or `777`), triggering the SSH client's self-defense mechanism.
- **Resolution:** Run `chmod 400 <key_name>.pem`.

### Problem 4: `Permission denied (publickey).` during SSH attempt

- **Root Cause:** You are using the correct key, but logging in with the wrong default username for the specific OS you provisioned.
- **Resolution:** Use the correct default user for your AMI:
  - **Amazon Linux:** `ssh -i key.pem ec2-user@<ip>`
  - **Ubuntu:** `ssh -i key.pem ubuntu@<ip>`
  - **Debian:** `ssh -i key.pem admin@<ip>`
  - **CentOS:** `ssh -i key.pem centos@<ip>`
