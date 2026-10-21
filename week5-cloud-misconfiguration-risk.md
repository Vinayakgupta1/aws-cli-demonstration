# WEEK 5 — CLOUD THREATS, VULNERABILITIES & RISK ASSESSMENT

---

# DAY 29 — What Is Cloud Misconfiguration?

# Phase 1: Deploy Common Cloud Misconfigurations via AWS Console & CLI

## Step 1: Create a Publicly Accessible S3 Bucket

* Log in to the AWS Management Console using administrator credentials.
* Search for **S3** and click **Create bucket**.
* Bucket name:

```
week5-misconfig-demo-2026-vinayak

```

* Under **Block Public Access settings for this bucket**:
* Uncheck **Block *all* public access**.
* Acknowledge the warning that the bucket will become public.


* Keep remaining settings default and click **Create bucket**.

---

## Step 2: Attach a Permissive Public Bucket Policy

* Open the created bucket `week5-misconfig-demo-2026-vinayak`.
* Go to the **Permissions** tab → **Bucket policy** → Click **Edit**.
* Select the **JSON** tab and paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::week5-misconfig-demo-2026-vinayak/*"
    }
  ]
}

```

* Replace `vinayak` with your actual bucket name.
* Click **Save changes**.

---

# Phase 2: Demonstrate the Impact of Misconfiguration

## Step 1: Upload Sensitive Test File

Using terminal and administrator credentials:

```bash
echo "CONFIDENTIAL: Financial Projections 2026" > confidential.txt

```

Upload it:

```bash
aws s3 cp confidential.txt \
s3://week5-misconfig-demo-2026-vinayak/confidential.txt

```

---

## Step 2: Retrieve the File Without Credentials

Open an unauthenticated terminal session or browser window (without AWS credentials):

```bash
curl -i https://week5-misconfig-demo-2026-vinayak.s3.amazonaws.com/confidential.txt

```

### Expected Output

```
HTTP/1.1 200 OK
Content-Type: text/plain

CONFIDENTIAL: Financial Projections 2026

```

This demonstrates:

```
Permissive Settings
       ↓
Unauthenticated Access
       ↓
Sensitive Data Leakage

```

---

# DAY 30 — Excessive Privileges & IAM Escalation

# Phase 1: Create an Overprivileged App Policy & Identity

## Step 1: Create the Overprivileged IAM Policy

* Search for **IAM** → **Policies → Create policy**.
* Select the **JSON** tab and paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ExcessiveAppAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "iam:AttachUserPolicy"
      ],
      "Resource": "*"
    }
  ]
}

```

* Click **Next**.
* Name:

```
Week5-AppService-Policy

```

* Click **Create policy**.

---

## Step 2: Create the Service Identity & CLI Credentials

* Go to **IAM → Users → Create user**.
* User name:

```
week5-app-service

```

* Leave console access disabled → Click **Next**.
* Choose **Attach policies directly** → Select `Week5-AppService-Policy`.
* Click **Next → Create user**.
* Open `week5-app-service` → **Security credentials** tab → **Access keys** → **Create access key**.
* Select **CLI** → Complete wizard → Copy key pair.

---

# Phase 2: Configure CLI for the App Service Identity

Open terminal:

```bash
aws configure --profile week5-app-service

```

Enter:

```
AWS Access Key ID: <app-service-access-key>
AWS Secret Access Key: <app-service-secret-key>
Default region name: us-east-1
Default output format: json

```

---

# Phase 3: Demonstrate Privilege Escalation Attack Path

Run an attempt to escalate privileges using the `iam:AttachUserPolicy` permission:

```bash
aws iam attach-user-policy \
--user-name week5-app-service \
--policy-arn arn:aws:iam::aws:policy/AdministratorAccess \
--profile week5-app-service

```

Now test administrative power:

```bash
aws iam list-users \
--profile week5-app-service

```

### Expected Output

```json
{
    "Users": [
        {
            "Path": "/",
            "UserName": "admin",
            "Arn": "arn:aws:iam::123456789012:root"
        },
        {
            "Path": "/",
            "UserName": "week5-app-service",
            "Arn": "arn:aws:iam::123456789012:user/week5-app-service"
        }
    ]
}

```

The attack flow is:

```
Low-Privilege Identity (week5-app-service)
       ↓
iam:AttachUserPolicy Granted
       ↓
Self-Attach AdministratorAccess Policy
       ↓
Full Cloud Account Compromise

```

---

# DAY 31 — Public Storage & Hardcoded Secrets

# Phase 1: Simulate Secret Exposure in Repository Code

Create a local code file containing embedded keys:

```bash
cat << 'EOF' > app_config.py
# Production App Credentials
AWS_ACCESS_KEY_ID = "AKIAIOSFODNN7EXAMPLE"
AWS_SECRET_ACCESS_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
DATABASE_URL = "postgres://admin:Password123!@db.internal.lab:5432/production"
EOF

```

---

# Phase 2: Automated Scanning for Cloud Secrets

Run `trufflehog` locally to detect exposed credentials:

```bash
trufflehog filesystem app_config.py --only-verified

```

### Expected Output

```
Found unverified result 🐷👇
Detector Type: AWS
Raw result: AKIAIOSFODNN7EXAMPLE
File: app_config.py
Line: 3

```

This demonstrates:

```
Hardcoded Secrets in Source Code
       ↓
Automated Scanner / Reconnaissance
       ↓
Credential Exposure

```

---

# DAY 32 — Insecure Cloud APIs & Burp Suite Testing

# Phase 1: Configure Burp Suite Proxy

* Launch **Burp Suite**.
* Ensure proxy listener is active at `127.0.0.1:8080`.
* Configure browser proxy settings to target `127.0.0.1:8080`.

---

# Phase 2: Intercept and Test Broken Object Level Authorization (BOLA)

* Open browser and navigate to target lab API:

```
http://api.lab.internal/v1/user/profile?id=101

```

* In Burp Suite, turn **Intercept ON**.
* Modify the request parameter from `id=101` (authenticated user) to `id=102` (victim user):

```http
GET /v1/user/profile?id=102 HTTP/1.1
Host: api.lab.internal
Authorization: Bearer <valid-token-for-user-101>

```

* Click **Forward**.

### Expected Output

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 102,
  "username": "victim_user",
  "email": "victim@lab.internal",
  "ssn": "XXX-XX-6789"
}

```

The vulnerability flow is:

```
Valid Auth Token (User 101)
       ↓
API Endpoint (Missing Authorization Object Check)
       ↓
Access Data of User 102 (BOLA / IDOR)

```

---

# DAY 33 — Logging, Monitoring & Encryption Failures

# Phase 1: Audit Unencrypted Cloud Resources

Using administrator credentials, query bucket server-side encryption settings:

```bash
aws s3api get-bucket-encryption \
--bucket week5-misconfig-demo-2026-vinayak

```

### Expected Output

```
An error occurred (ServerSideEncryptionConfigurationNotFoundError) when calling the GetBucketEncryption operation: The server side encryption configuration was not found

```

---

# Phase 2: Audit Logging Status across Services

Check if active CloudTrail logging exists:

```bash
aws cloudtrail describe-trails

```

### Expected Output

```json
{
    "trailList": []
}

```

This demonstrates visibility failure:

```
Unencrypted Data + Disabled CloudTrail Logs
       ↓
Data Exfiltration Attack
       ↓
Zero Detection / Zero Forensic Visibility

```

---

# DAY 34 — Automated Cloud Security Scanning

# Phase 1: Execute ScoutSuite Architecture Assessment

Run an automated security audit of the AWS environment:

```bash
scout aws --profile default --regions us-east-1

```

---

# Phase 2: Execute Prowler Security Baseline Assessment

Run Prowler targeting IAM and S3 security controls:

```bash
prowler aws --services s3 iam -M json-ocsf

```

### Expected Output

```
[FAIL] Ensure IAM policies that allow full "*:*" administrative privileges are restricted
[FAIL] Ensure S3 bucket access logging is enabled
[FAIL] Ensure S3 Buckets are configured with Block Public Access

```

---

# DAY 35 — Risk Assessment & Finding Framework

# Phase 1: The Risk Ranking Pipeline

Apply this systematic model to evaluate findings:

```
Vulnerability Finding
       ↓
Identify Asset Criticality
       ↓
Assess Potential Impact
       ↓
Assess Exploitation Likelihood
       ↓
Determine Overall Risk (Critical / High / Medium / Low)
       ↓
Define Remediation Action

```

---

# Phase 2: Cloud Security Finding Template

Use this structured format for risk documentation:

| Field | Details |
| --- | --- |
| **Finding ID** | `FINDING-AWS-2026-001` |
| **Title** | Public S3 Bucket Containing Sensitive Data |
| **Affected Asset** | `arn:aws:s3:::week5-misconfig-demo-2026-vinayak` |
| **Severity** | **CRITICAL** |
| **Impact** | Unauthenticated exposure of sensitive financial records to the public internet. |
| **Likelihood** | High — Bucket policy explicitly permits global read operations. |
| **Remediation** | Enable **S3 Block Public Access** and enforce default KMS encryption. |

---
