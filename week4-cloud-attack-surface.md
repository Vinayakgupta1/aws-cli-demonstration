# WEEK 4 — CLOUD ATTACK SURFACE

## DAY 22 — Cloud Reconnaissance

# Phase 1: Passive & Active Asset Discovery

## Step 1: Enumerate DNS & Subdomains

* Perform DNS queries to resolve target cloud-hosted domain endpoints:

```bash
dig +short A app.example.com

```

* Query CNAME records to identify underlying AWS services (e.g., CloudFront, ELB):

```bash
dig CNAME app.example.com

```

---

## Step 2: Enumerate Public IP Ranges & Load Balancers

* Search for active EC2 instances assigned public IP addresses:

```bash
aws ec2 describe-instances \
  --query "Reservations[*].Instances[*].{ID:InstanceId,PublicIP:PublicIpAddress,State:State.Name}" \
  --output table

```

* List active public load balancers:

```bash
aws elbv2 describe-load-balancers \
  --query "LoadBalancers[*].{Name:LoadBalancerName,DNS:DNSName,Scheme:Scheme}" \
  --output table

```

---

## Step 3: Verify the Reconnaissance Mapping

The resulting discovery flow is:

```
Target Domain
      │
      ↓
DNS Enumeration
      │
      ↓
Cloud CNAME / Infrastructure
      │
      ├── Public EC2 IP Addresses
      └── Load Balancer DNS Endpoints

```

---

# DAY 23 — Cloud Identity Attacks

# Phase 1: Audit Weak & Overprivileged Policies

## Step 1: Inspect User Identity Policies

* Open **IAM → Users → week4-lab-user**.
* Select **Permissions**.
* Review attached policies for wildcard actions (`*`) or overly broad service access.

---

## Step 2: Retrieve and Analyze Policy JSON via AWS CLI

Run:

```bash
aws iam get-policy-version \
  --policy-arn arn:aws:iam::123456789012:policy/Week4-Permissive-Policy \
  --version-id v1

```

### Expected Output

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ExcessivePermissions",
      "Effect": "Allow",
      "Action": [
        "iam:*",
        "s3:*"
      ],
      "Resource": "*"
    }
  ]
}

```

---

## Step 3: Demonstrate Credential Abuse Risk

The identity exploitation architecture is:

```
Exposed / Weak Access Key
            │
            ↓
    AWS API Execution
            │
            ↓
  Overprivileged Policy
            │
            ├── Full IAM Control (iam:*)
            └── Full S3 Control (s3:*)

```

---

# DAY 24 — S3 Attacks

# Phase 1: Audit Vulnerable Bucket Access

## Step 1: Check Bucket Public Access Block Status

Run:

```bash
aws s3api get-public-access-block \
  --bucket week4-s3-lab-2026-vinayak

```

### Expected Output

```json
{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": false,
        "IgnorePublicAcls": false,
        "BlockPublicPolicy": false,
        "RestrictPublicBuckets": false
    }
}

```

---

## Step 2: Test Unauthenticated Data Access

Run anonymously (without AWS credentials):

```bash
curl -i https://week4-s3-lab-2026-vinayak.s3.amazonaws.com/sensitive-data.txt

```

### Expected Output

```
HTTP/1.1 200 OK
CONFIDENTIAL_API_KEY=sk_live_9988776655

```

---

## Step 3: Apply S3 Bucket Remediation

* Search for **S3** → Select `week4-s3-lab-2026-vinayak`.
* Select **Permissions**.
* Under **Block public access (bucket settings)**, click **Edit**.
* Enable **Block *all* public access**.
* Click **Save changes**.

---

# DAY 25 — Network Attack Surface

# Phase 1: Audit Open Security Groups & Exposed Services

## Step 1: Identify Inbound Rules Open to `0.0.0.0/0`

* Search for **VPC** → **Security groups**.
* Filter or query rules allowing global inbound access:

```bash
aws ec2 describe-security-groups \
  --filters Name=ip-permission.cidr,Values='0.0.0.0/0' \
  --query "SecurityGroups[*].{ID:GroupId,Name:GroupName,Permissions:IpPermissions}" \
  --output json

```

---

## Step 2: Revoke High-Risk Public Access

Remove unrestricted SSH access (`0.0.0.0/0`) from the web security group:

```bash
aws ec2 revoke-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

```

---

## Step 3: Verify Security Group Architecture

The hardened security group architecture becomes:

```
Internet (0.0.0.0/0)
       │
       ├── HTTP (80)  ──> ALLOWED
       ├── HTTPS (443) ──> ALLOWED
       │
       X
  SSH (Port 22)
(Restricted to Admin CIDR / SSM)

```

---

# DAY 26 — EC2 Metadata & IMDS

# Phase 1: IMDS Architecture & Version Comparison

## Step 1: Inspect Instance Metadata Configuration

Run:

```bash
aws ec2 describe-instances \
  --instance-ids i-0123456789abcdef0 \
  --query "Reservations[*].Instances[*].MetadataOptions"

```

---

## Step 2: Enforce IMDSv2 via AWS CLI

Run:

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-0123456789abcdef0 \
  --http-tokens required \
  --http-endpoint enabled

```

---

## Step 3: Verify IMDS Enforcement

The resulting metadata control structure is:

```
IMDSv1 (Insecure)
  - Direct HTTP GET request
  - No token required
  - Vulnerable to simple SSRF
         │
         ↓
IMDSv2 (Secure)
  - PUT request to acquire Session Token
  - Token required in X-aws-ec2-metadata-token header
  - Mitigates standard GET-based SSRF

```

---

# DAY 27 — SSRF → Cloud Credential Exposure

# Phase 1: Conceptual Mechanics of the SSRF Attack Chain

## Step 1: Analyze the Exploitation Path

* An attacker identifies a vulnerable web application parameter accepting external URLs.
* The web application processes the request server-side and queries internal resources.
* The target request routes to the link-local metadata address:

```
http://169.254.169.254/latest/meta-data/iam/security-credentials/

```

---

## Step 2: Review Temporary Role Credentials Format

When IMDSv1 is queried via SSRF, the metadata endpoint returns temporary credentials:

```json
{
  "Code": "Success",
  "AccessKeyId": "ASIA...",
  "SecretAccessKey": "wJalrXU...",
  "Token": "IQoJb3JpZ2luX2VjE...",
  "Expiration": "2026-10-21T18:00:00Z"
}

```

---

## Step 3: Verify the SSRF Attack Flow

The complete cloud breach flow is:

```
Web Application
       │
       ↓
     SSRF
       │
       ↓
EC2 Metadata (http://169.254.169.254)
       │
       ↓
Temporary Credentials (IAM Role)
       │
       ↓
AWS API Access (S3, EC2, IAM)

```

---

# DAY 28 — Vulnerable Cloud Challenge

# Phase 1: Environment Security Audit

## Step 1: Execute Asset & Attack Path Analysis

Students perform hands-on reconnaissance across the challenge environment to map weaknesses:

1. **Reconnaissance:** Identify active public web server endpoints.
2. **Metadata Inspection:** Check IMDS version enforcement on compute instances.
3. **IAM Review:** Audit attached IAM role permissions for broad resource access.
4. **Storage Review:** Identify S3 bucket policies with missing public blocks.

---

## Step 2: Document Attack Surface Evidence

The attack chain mapped during the challenge exercise is:

```
Web Server Endpoint
       │
       ↓
Unsecured IMDSv1 Access
       │
       ↓
Stolen IAM Role Credentials
       │
       ↓
Overprivileged S3 Read Policy
       │
       ↓
Public S3 Bucket Exfiltration

```

---

# Phase 2: Cloud Attack Surface Report Deliverable

```markdown
# CLOUD ATTACK SURFACE REPORT

**Target Environment:** Vulnerable Cloud Challenge Lab  
**Assessor:** Security Analyst / Student  
**Date:** October 21, 2026  

---

## Executive Summary
A comprehensive security review of the target environment revealed critical exposure points. An external actor can discover public web endpoints, exploit IMDSv1 to extract temporary IAM role credentials, and leverage excessive IAM permissions to exfiltrate data from unencrypted S3 storage buckets.

---

## Key Vulnerability Findings

### Finding 1: Unrestricted Security Group Ingress (SSH)
- **Severity:** High
- **Description:** Security group `sg-0123456789abcdef0` allows inbound SSH (Port 22) access from `0.0.0.0/0`.
- **Remediation:** Remove global ingress and restrict access to management IP blocks or AWS Systems Manager Session Manager.

### Finding 2: IMDSv1 Enabled on Compute Instances
- **Severity:** High
- **Description:** EC2 instance `i-0123456789abcdef0` permits unauthenticated IMDSv1 metadata access, allowing credential theft via SSRF.
- **Remediation:** Require IMDSv2 (`HttpTokens=required`).

### Finding 3: Publicly Accessible S3 Bucket
- **Severity:** Critical
- **Description:** Bucket `week4-s3-lab-2026-vinayak` lacks Block Public Access controls and permits global read access.
- **Remediation:** Enable S3 Block Public Access settings globally across the bucket.

```
