 # WEEK 2 — IAM & IDENTITY SECURITY

 ## DAY 8 — IAM Fundamentals

 # Phase 1: Create the IAM Lab Resources via the AWS Console

 ## Step 1: Create the S3 Bucket

 - Log in to the AWS Management Console using your administrator/training credentials.
- Search for **S3**.
- Click **Create bucket**.
- Enter:

```
week2-iam-lab-2026-vinayak
```

 - Keep **Block all public access** enabled.
- Keep the remaining settings at their defaults.
- Click **Create bucket**.

---

 ## Step 2: Create a Custom IAM Policy

 - Search for **IAM**.
- Choose **Policies → Create policy**.
- Select the **JSON** tab.
- Paste:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowLabBucketRead",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::week2-iam-lab-2026-vinayak"
    },
    {
      "Sid": "AllowLabBucketObjectRead",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::week2-iam-lab-2026-vinayak/*"
    },
    {
      "Sid": "AllowEC2Inspection",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeSecurityGroups"
      ],
      "Resource": "*"
    }
  ]
}
```

 - Replace `vinayak` with your actual bucket name.
- Click **Next**.
- Name the policy:

```
Week2-Developer-Policy
```

 - Click **Create policy**.

---

 ## Step 3: Create an IAM Group

 - In IAM, choose **User groups → Create group**.
- Enter:

```
Week2-Developers
```

 - Under **Attach permissions policies**, select:

```
Week2-Developer-Policy
```

 - Click **Create group**.

---

 ## Step 4: Create an IAM User

 - Choose **Users → Create user**.
- Enter:

```
week2-developer
```

 - For this lab, leave console access disabled.
- Click **Next**.
- Select **Add user to group**.
- Select:

```
Week2-Developers
```

 - Click **Next → Create user**.

---

 ## Step 5: Verify the IAM Architecture

 The resulting structure should be:

```
week2-developer
       │
       ↓
Week2-Developers
       │
       ↓
Week2-Developer-Policy
       │
       ├── s3:ListBucket
       ├── s3:GetObject
       ├── ec2:DescribeInstances
       └── ec2:DescribeSecurityGroups
```

---

 # DAY 9 — Authentication vs Authorization

 # Phase 1: Create CLI Credentials

 - Open **IAM → Users → week2-developer**.
- Select **Security credentials**.
- Find **Access keys**.
- Click **Create access key**.
- Select **Command Line Interface (CLI)**.
- Complete the wizard.
- Securely copy the Access Key ID and Secret Access Key.

 > Use these credentials only for this isolated lab. Never place the secret key in source code, Git repositories, screenshots, or chat.

---

 # Phase 2: Configure the AWS CLI

 Open your terminal:

```
aws configure --profile week2-developer
```

 Enter:

```
AWS Access Key ID: <your-access-key>
AWS Secret Access Key: <your-secret-key>
Default region name: us-east-1
Default output format: json
```

---

 # Phase 3: Demonstrate Authentication

 Run:

```
aws sts get-caller-identity \
--profile week2-developer
```

 ### Expected Output

```
{
    "UserId": "AIDA...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/week2-developer"
}
```

 This demonstrates:

```
Credentials
    ↓
Authentication
    ↓
AWS identifies the IAM user
```

---

 # Phase 4: Demonstrate Authorization

 Run:

```
aws s3 ls \
s3://week2-iam-lab-2026-vinayak \
--profile week2-developer
```

 ### Expected Output

 Success.

 Now try an operation that isn't allowed:

```
aws s3api delete-bucket \
--bucket week2-iam-lab-2026-vinayak \
--profile week2-developer
```

 ### Expected Output

```
AccessDenied
```

 The complete flow is:

```
Authentication
      ↓
Who are you?
      ↓
week2-developer
      ↓
Authorization
      ↓
What can you do?
      ↓
IAM Policy
      ↓
Allowed / Denied
```

---

 # DAY 10 — IAM Policies

 # Phase 1: Create an S3 Test File

 Using your administrator/training credentials:

```
echo "IAM Week 2 Test File" > test.txt
```

 Upload it:

```
aws s3 cp test.txt \
s3://week2-iam-lab-2026-vinayak/test.txt
```

---

 # Phase 2: Test `s3:GetObject`

 Using the developer profile:

```
aws s3 cp \
s3://week2-iam-lab-2026-vinayak/test.txt \
downloaded.txt \
--profile week2-developer
```

 ### Expected Output

```
download: s3://.../test.txt
```

 This demonstrates:

```
Effect = Allow
Action = s3:GetObject
Resource = specific bucket object
```

---

 # Phase 3: Test an Unauthorized Action

 Run:

```
aws s3api delete-object \
--bucket week2-iam-lab-2026-vinayak \
--key test.txt \
--profile week2-developer
```

 ### Expected Output

```
AccessDenied
```

---

 # Phase 4: Test EC2 Read Permissions

 Run:

```
aws ec2 describe-instances \
--profile week2-developer \
--region us-east-1
```

 ### Expected Output

 Success, for example:

```
{
    "Reservations": []
}
```

 or a list of existing instances.

 Now test a write-level action using a fake instance ID:

```
aws ec2 terminate-instances \
--instance-ids i-00000000000000000 \
--profile week2-developer \
--region us-east-1
```

 ### Expected Output

```
AccessDenied
```

---

 # DAY 11 — LEAST PRIVILEGE

 # Phase 1: Create an Overprivileged Policy

 - Open **IAM → Policies → Create policy**.
- Select **JSON**.
- Paste:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "OverPrivilegedDeveloper",
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "ec2:*",
        "lambda:*"
      ],
      "Resource": "*"
    }
  ]
}
```

 - Name it:

```
Week2-Developer-OverPrivileged
```

 - Click **Create policy**.

---

 # Phase 2: Attach the Excessive Policy

 - Open **IAM → Users → week2-developer**.
- Choose **Add permissions**.
- Attach:

```
Week2-Developer-OverPrivileged
```

---

 # Phase 3: Demonstrate Excessive Access

 Run:

```
aws s3api list-buckets \
--profile week2-developer
```

 Then:

```
aws ec2 describe-instances \
--profile week2-developer \
--region us-east-1
```

 Then:

```
aws lambda list-functions \
--profile week2-developer \
--region us-east-1
```

 The identity now has considerably more access than the developer requires.

---

 # Phase 4: Remove Excessive Permissions

 Remove:

```
s3:*
ec2:*
lambda:*
```

 Restore the limited policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ReadWrite",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::week2-iam-lab-2026-vinayak"
    },
    {
      "Sid": "S3ObjectReadWrite",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::week2-iam-lab-2026-vinayak/*"
    },
    {
      "Sid": "EC2Inspection",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeSecurityGroups"
      ],
      "Resource": "*"
    }
  ]
}
```

---

 # Phase 5: Verify Least Privilege

 Allowed:

```
aws s3 ls \
s3://week2-iam-lab-2026-vinayak \
--profile week2-developer
```

 Expected:

```
SUCCESS
```

 Allowed:

```
aws ec2 describe-instances \
--profile week2-developer \
--region us-east-1
```

 Expected:

```
SUCCESS
```

 Blocked:

```
aws s3api delete-bucket \
--bucket week2-iam-lab-2026-vinayak \
--profile week2-developer
```

 Expected:

```
AccessDenied
```

---

 # DAY 12 — MFA, ACCESS KEYS & TEMPORARY CREDENTIALS

 # Phase 1: Inspect Access Keys

 - Open **IAM → Users → week2-developer**.
- Select **Security credentials**.
- Locate **Access keys**.
- Verify the access key exists.

 The architecture is:

```
Access Key ID
       +
Secret Access Key
       ↓
AWS CLI
       ↓
AWS API
```

---

 # Phase 2: Demonstrate Credential Authentication

 Run:

```
aws sts get-caller-identity \
--profile week2-developer
```

 Expected:

```
{
    "Arn": "arn:aws:iam::<ACCOUNT-ID>:user/week2-developer"
}
```

---

 # Phase 3: Configure MFA

 For the appropriate console-enabled training identity:

 - Open **IAM → Users**.
- Select the user.
- Choose **Security credentials**.
- Locate **Multi-factor authentication (MFA)**.
- Click **Assign MFA device**.
- Choose an appropriate authenticator method.
- Complete the MFA setup.

 The authentication flow becomes:

```
Password
   +
MFA
   ↓
Authentication
```

---

 # Phase 4: Create an IAM Role for EC2

 - Open **IAM → Roles → Create role**.
- Select **AWS service**.
- Select **EC2**.
- Attach the limited EC2 inspection policy.
- Name the role:

```
Week2-EC2-DeveloperRole
```

 The architecture becomes:

```
EC2
 │
 ↓
IAM Role
 │
 ↓
Temporary Credentials
 │
 ↓
AWS API
```

 Instead of:

```
EC2
 │
 ↓
Hard-coded Access Key
 │
 ↓
AWS API
```

---

 # DAY 13 — IAM POLICY EVALUATION

 # Phase 1: Demonstrate Explicit Allow

 Use:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Read",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::week2-iam-lab-2026-vinayak/*"
    }
  ]
}
```

 Test:

```
aws s3 cp \
s3://week2-iam-lab-2026-vinayak/test.txt \
test-download.txt \
--profile week2-developer
```

 Expected:

```
SUCCESS
```

---

 # Phase 2: Demonstrate Implicit Deny

 Run:

```
aws s3api delete-object \
--bucket week2-iam-lab-2026-vinayak \
--key test.txt \
--profile week2-developer
```

 Expected:

```
AccessDenied
```

 Because there is no applicable Allow for `s3:DeleteObject`.

```
No Allow
   ↓
Implicit Deny
   ↓
AccessDenied
```

---

 # Phase 3: Demonstrate Explicit Deny

 Create a policy containing:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Actions",
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::week2-iam-lab-2026-vinayak",
        "arn:aws:s3:::week2-iam-lab-2026-vinayak/*"
      ]
    },
    {
      "Sid": "DenyDelete",
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::week2-iam-lab-2026-vinayak/*"
    }
  ]
}
```

 Test:

```
aws s3api delete-object \
--bucket week2-iam-lab-2026-vinayak \
--key test.txt \
--profile week2-developer
```

 Even though:

```
s3:*
```

 allows the action, the explicit Deny wins:

```
Allow s3:*
      +
Deny s3:DeleteObject
      ↓
DENY
```

---

 # Phase 4: Demonstrate the Policy Evaluation Flow

 Use this flow while explaining the result:

```
AWS Request
     ↓
Authentication
     ↓
Policy Evaluation
     ↓
Explicit Deny?
   /       \
 YES       NO
  ↓         ↓
DENY    Applicable Allow?
            /       \
          YES        NO
           ↓          ↓
         ALLOW     IMPLICIT DENY
```

---
