# SOP: AWS Key Pair Provisioning & Cloud Authentication

## 1️⃣ The Objective (The "Why")

To establish secure, passwordless access to AWS EC2 instances. Cloud environments enforce Asymmetric Cryptography to prevent brute-force attacks. This operation provisions a public/private key pair via Infrastructure as Code (IaC) tools, securely storing the public key in the AWS database while isolating the private key on the local engineering machine.

## 2️⃣ Target Infrastructure (The "Where")

- **Cloud Provider**: Amazon Web Services (AWS)
- **Service**: Elastic Compute Cloud (EC2)
- **Core Tool**: AWS CLI (`aws ec2`)
- **Cryptography**: RSA (Rivest–Shamir–Adleman)

## 3️⃣ Execution (The "How")

### Step 1: Generate the Key Pair via AWS CLI

Use the AWS API to generate the cryptographic math. We must use specific queries to extract only the raw private key text and redirect it into a local `.pem` file.

```bash
aws ec2 create-key-pair --key-name datacenter-kp --key-type rsa --query 'KeyMaterial' --output text > datacenter-kp.pem
```

**Flag Breakdown:**
- `--key-name`: The logical name AWS will use to store the Public Key lock in its database.
- `--key-type`: Specifies the cryptographic algorithm (`rsa` or `ed25519`).
- `--query 'KeyMaterial'`: Filters the massive JSON response to output only the private key string.
- `> datacenter-kp.pem`: Redirects the standard output into a physical file on the local machine.

### Step 2: Enforce the "400 Rule" (Local Security)

Linux SSH clients possess a built-in failsafe: if a private key file is readable by other users on the local machine, the connection is instantly aborted. You must lock the file down to owner-read-only permissions immediately.

```bash
chmod 400 datacenter-kp.pem
```

## 4️⃣ Verification & Troubleshooting (The "Proof")

### SRE Verification

Do not assume the AWS API accepted the command. Query the AWS database to mathematically prove the Public Key was registered.

```bash
aws ec2 describe-key-pairs --key-names datacenter-kp
```

**Expected Output:** A JSON block containing the `KeyPairId` and fingerprint.

### Common Troubleshooting Vectors

#### Problem 1: `An error occurred (InvalidKeyPair.Duplicate)... The keypair already exists`
- **Root Cause**: AWS already has a public key registered under this name. AWS does not save your private key, so if you lose your local `.pem` file, you cannot just run the create command again.
- **Resolution**: You must explicitly delete the orphaned lock from the AWS database before generating a new one:
  ```bash
  aws ec2 delete-key-pair --key-name datacenter-kp
  ```

#### Problem 2: `An error occurred (InvalidKeyPair.NotFound)... The key pair 'datacenter-kp.pem' does not exist`
- **Root Cause**: This occurs during the verification (describe) step if you append the `.pem` extension to the key name.
- **Resolution**: AWS does not know about the `.pem` file on your local hard drive. It only knows the logical database name. Omit `.pem` and query using strictly the `--key-name` (e.g., `datacenter-kp`).

#### Problem 3: `WARNING: UNPROTECTED PRIVATE KEY FILE!`
- **Root Cause**: You skipped Step 2. The local permissions are too broad (e.g., `644`).
- **Resolution**: Run `chmod 400 <key_name>.pem`.
