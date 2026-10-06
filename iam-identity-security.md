# WEEK 2 — IAM & IDENTITY SECURITY: PRACTICAL AWS LABS

 The following labs use **one continuous AWS environment** across Days 8–14. Students progressively create an IAM identity, assign permissions, test authorization, introduce excessive privileges, reduce them using least privilege, examine MFA and credentials, understand policy evaluation, and finally perform an IAM security assessment.

 > **Lab safety:** Perform these exercises only in a dedicated training AWS account. Do not use the AWS root account for day-to-day work. The access key exercise is intentionally included for educational purposes; for real environments, prefer IAM roles, IAM Identity Center, and temporary credentials.

---

 # DAY 8 — IAM FUNDAMENTALS

 ## Phase 1: Set Up IAM and Resources via the AWS Console

 ### Step 1: Create the S3 Bucket

 - Log in to the AWS Management Console using your administrator/training credentials and search for **S3**.
- Choose **Create bucket**.
- Enter the bucket name:

```
week2-iam-lab-2026-<unique-name>
```

 - Keep **Block all public access** enabled.
- Keep the remaining settings at their defaults.
- Click **Create bucket**.

 > **Expected Result:** The S3 bucket is created and remains private.

---

 ### Step 2: Create an IAM Policy

 - Search for **IAM**.
- In the left navigation pane, choose **Policies → Create policy**.
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
      "Resource": "arn:aws:s3:::week2-iam-lab-2026-<unique-name>"
    },
    {
      "Sid": "AllowLabBucketObjectRead",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::week2-iam-lab-2026-<unique-name>/*"
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

 - Replace `<unique-name>` with the actual bucket name.
- Click **Next**.
- Name the policy:

```
Week2-Developer-ReadOnly
```

 - Click **Create policy**.

 > **Security Concept:** Notice that `s3:ListBucket` uses the bucket ARN, while `s3:GetObject` uses the object ARN (`/*`). This demonstrates resource-level permissions.

---

 ### Step 3: Create the Developers Group

 - In the IAM sidebar, choose **User groups → Create group**.
- Enter:

```
Week2-Developers
```

 - Under **Attach permissions policies**, select:

```
Week2-Developer-ReadOnly
```

 - Click **Create group**.

---

 ### Step 4: Create the Developer User

 - Choose **Users → Create user**.
- Enter:

```
week2-developer
```

 - For this initial lab, leave console access disabled.
- Click **Next**.
- Select **Add user to group**.
- Select:

```
Week2-Developers
```

 - Click **Next → Create user**.

---

 ## Phase 2: Understand the IAM Architecture

 The resulting architecture should look like:

```
week2-developer
       │
       ↓
Week2-Developers
       │
       ↓
Week2-Developer-ReadOnly
       │
       ├── S3 Read
       │
       └── EC2 Describe
```

 ### Identify Each Component

 | Component | Purpose |
| --- | --- |
| User | Represents an identity |
| Group | Organizes users |
| Policy | Defines permissions |
| Permission | Allows/denies an AWS action |
| Resource | AWS object being accessed |

---

 ## Phase 3: IAM Fundamentals Exercise

 Ask students to answer:

 1. Who is the identity?
2. What group does the identity belong to?
3. What policy is attached?
4. Which S3 actions are allowed?
5. Which EC2 actions are allowed?
6. Is the user allowed to terminate an EC2 instance?

 ### Expected Answers

```
Identity:
week2-developer

Group:
Week2-Developers

Policy:
Week2-Developer-ReadOnly

S3:
ListBucket
GetObject

EC2:
DescribeInstances
DescribeSecurityGroups

TerminateInstances:
DENIED
```

---

 # DAY 9 — AUTHENTICATION VS AUTHORIZATION

 ## Phase 1: Configure CLI Authentication

 For this controlled lab, create a dedicated CLI access key for `week2-developer`.

 - Open **IAM → Users → week2-developer**.
- Select **Security credentials**.
- Under **Access keys**, choose **Create access key**.
- Select **Command Line Interface (CLI)**.
- Complete the wizard.
- Copy the credentials temporarily to a secure location.

 > **Important:** The secret access key is displayed only when it is created. Never put it in GitHub, screenshots, chat messages, source code, or a public document.

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

 Verify authentication:

```
aws sts get-caller-identity --profile week2-developer
```

 ### Expected Output

```
{
    "UserId": "AIDA...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/week2-developer"
}
```

 ### Security Concept

 This demonstrates:

```
Authentication
      ↓
"Who are you?"
      ↓
AWS identifies week2-developer
```

---

 # Phase 3: Demonstrate Authorization

 Run:

```
aws s3 ls s3://week2-iam-lab-2026-<unique-name> \
  --profile week2-developer
```

 ### Expected Output

 Success.

 The bucket may be empty, so there may be no objects listed.

 Now try:

```
aws s3api delete-bucket \
  --bucket week2-iam-lab-2026-<unique-name> \
  --profile week2-developer
```

 ### Expected Output

```
An error occurred (AccessDenied) when calling the DeleteBucket operation
```

 This demonstrates:

```
Authentication
      ↓
"I am week2-developer"

Authorization
      ↓
"Can week2-developer delete this bucket?"

      ↓

NO
      ↓
AccessDenied
```

---

 # DAY 10 — IAM POLICIES

 ## Phase 1: Understand IAM Policy Anatomy

 Create the following policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Read",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::week2-iam-lab-2026-<unique-name>/*"
    }
  ]
}
```

 Explain:

```
Version
   ↓
Policy language version

Statement
   ↓
Permission rule

Effect
   ↓
Allow / Deny

Action
   ↓
AWS API operation

Resource
   ↓
Target AWS resource

Condition
   ↓
Optional restrictions
```

---

 # Phase 2: Test `Allow`

 Upload a file to the bucket using your administrator/training account:

```
echo "IAM Week 2 Test File" > test.txt
```

 Upload it:

```
aws s3 cp test.txt \
s3://week2-iam-lab-2026-<unique-name>/test.txt
```

 Now use the developer profile:

```
aws s3 cp \
s3://week2-iam-lab-2026-<unique-name>/test.txt \
downloaded.txt \
--profile week2-developer
```

 ### Expected Output

```
download: s3://.../test.txt to ./downloaded.txt
```

---

 # Phase 3: Test a Restricted Action

 Try:

```
aws s3 rm \
s3://week2-iam-lab-2026-<unique-name>/test.txt \
--profile week2-developer
```

 ### Expected Output

```
AccessDenied
```

 Students should understand:

```
s3:GetObject
      ↓
Explicit Allow
      ↓
SUCCESS

s3:DeleteObject
      ↓
No Allow
      ↓
Implicit Deny
```

---

 # Phase 4: Add EC2 Inspection Permissions

 Update the policy with:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Read",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::week2-iam-lab-2026-<unique-name>",
        "arn:aws:s3:::week2-iam-lab-2026-<unique-name>/*"
      ]
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

 Test:

```
aws ec2 describe-instances \
--profile week2-developer \
--region us-east-1
```

 ### Expected Output

 Success:

```
{
    "Reservations": []
}
```

 or a JSON response containing existing EC2 instances.

 Now test:

```
aws ec2 terminate-instances \
--instance-ids i-0123456789abcdef0 \
--profile week2-developer \
--region us-east-1
```

 ### Expected Output

```
AccessDenied
```

 > Use a fake/nonexistent instance ID for this test. Do not terminate a real training instance merely to demonstrate authorization failure.

---

 # DAY 11 — LEAST PRIVILEGE

 ## Phase 1: Create an Overprivileged Policy

 Create a deliberately excessive policy for the exercise.

 - IAM → Policies → Create policy.
- Select **JSON**.
- Enter:

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
        "lambda:*",
        "iam:ListUsers",
        "iam:ListRoles"
      ],
      "Resource": "*"
    }
  ]
}
```

 Name it:

```
Week2-Developer-OverPrivileged
```

---

 # Phase 2: Attach the Overprivileged Policy

 Attach it to:

```
week2-developer
```

 or to a dedicated group:

```
Week2-OverPrivileged-Developers
```

 The resulting architecture is:

```
Developer
    ↓
S3 *
EC2 *
Lambda *
```

 Ask:

 > Does the developer really need all of these permissions?

---

 # Phase 3: Demonstrate Excessive Permissions

 Test:

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

 All may succeed because the identity has excessive permissions.

---

 # Phase 4: Reduce Permissions

 Remove:

```
s3:*
ec2:*
lambda:*
```

 Replace them with the actual business requirement:

```
S3:
- s3:ListBucket
- s3:GetObject
- s3:PutObject

EC2:
- ec2:DescribeInstances
- ec2:DescribeSecurityGroups
```

 The final architecture becomes:

```
Developer
    │
    ├── S3: Limited access
    │
    └── EC2: Read/inspection access
```

 ### Security Principle

```
More permissions
       ↓
Larger attack surface
       ↓
Greater impact if credentials are compromised

Least privilege
       ↓
Smaller attack surface
       ↓
Reduced potential impact
```

---

 # DAY 12 — MFA, ACCESS KEYS & TEMPORARY CREDENTIALS

 ## Phase 1: Understand Access Keys

 The developer's programmatic identity is represented by:

```
Access Key ID
       +
Secret Access Key
       ↓
AWS API Authentication
```

 Run:

```
aws sts get-caller-identity \
--profile week2-developer
```

 Explain that AWS is using the configured credentials to authenticate the request.

---

 # Phase 2: Demonstrate Credential Exposure Risk

 Create a demonstration file:

```
echo "AWS_SECRET_ACCESS_KEY=DEMO_SECRET" > exposed-demo.txt
```

 Explain:

```
Developer laptop
      ↓
Credential stored insecurely
      ↓
Malware / accidental upload / Git leak
      ↓
Credential exposure
      ↓
Unauthorized AWS access
```

 Do **not** use or publish a real secret.

---

 # Phase 3: MFA

 For a human console identity:

 - Open **IAM → Users**.
- Select the appropriate training user.
- Choose **Security credentials**.
- Locate **Multi-factor authentication (MFA)**.
- Choose **Assign MFA device**.
- Follow the wizard using an appropriate authenticator device.

 Explain:

```
Password
   +
MFA
   ↓
Stronger authentication
```

 ### Discussion

 Ask students:

 > If an attacker steals only the password, what additional barrier does MFA provide?

 Then explain that MFA is an authentication control; it doesn't replace authorization.

---

 # Phase 4: Temporary Credentials with IAM Roles

 Create an IAM role:

```
Week2-EC2-DeveloperRole
```

 Give the role only the required read permissions.

 The architecture becomes:

```
EC2 Instance
      │
      ↓
IAM Role
      │
      ↓
Temporary Credentials
      │
      ↓
AWS Services
```

 Compare this with:

```
EC2 Instance
      │
      ↓
Hard-coded Access Key
      │
      ↓
AWS Services
```

 ### Security Conclusion

```
Long-lived credentials
        ↓
Higher exposure risk

Temporary credentials
        ↓
Automatic expiration
        ↓
Reduced credential lifetime
```

---

 # DAY 13 — IAM POLICY EVALUATION

 ## Phase 1: Demonstrate Explicit Allow

 Create:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowRead",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::week2-iam-lab-2026-<unique-name>/*"
    }
  ]
}
```

 Test:

```
aws s3 cp \
s3://week2-iam-lab-2026-<unique-name>/test.txt \
test-download.txt \
--profile week2-developer
```

 Expected:

```
SUCCESS
```

---

 # Phase 2: Demonstrate Implicit Deny

 Try:

```
aws s3api delete-object \
--bucket week2-iam-lab-2026-<unique-name> \
--key test.txt \
--profile week2-developer
```

 Expected:

```
AccessDenied
```

 Explain:

```
Is there an Allow?
       ↓
NO
       ↓
Implicit Deny
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
        "arn:aws:s3:::week2-iam-lab-2026-<unique-name>",
        "arn:aws:s3:::week2-iam-lab-2026-<unique-name>/*"
      ]
    },
    {
      "Sid": "ExplicitlyDenyDelete",
      "Effect": "Deny",
      "Action": [
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::week2-iam-lab-2026-<unique-name>/*"
    }
  ]
}
```

 Now:

```
Allow:
s3:*

        +

Deny:
s3:DeleteObject

        ↓

s3:DeleteObject = DENIED
```

 This demonstrates the fundamental IAM rule:

 > **An explicit Deny overrides an Allow.**

---

 # Phase 4: Build the IAM Policy Evaluation Flowchart

 Students should create this flowchart in their PPT:

```
                AWS Request
                     │
                     ↓
              Authentication
                     │
                     ↓
              Policy Evaluation
                     │
                     ↓
             Explicit Deny?
               /          \
             YES            NO
              │              │
              ↓              ↓
            DENY         Is there an
                         applicable Allow?
                          /          \
                        YES           NO
                         │             │
                         ↓             ↓
                       ALLOW        DENY
                                  (Implicit)
```

---

 # DAY 14 — IAM SECURITY CHALLENGE

 # Phase 1: Create the Intentionally Insecure Environment

 Create an IAM policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "IntentionallyOverPrivileged",
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "ec2:*",
        "lambda:*",
        "iam:ListUsers",
        "iam:ListRoles"
      ],
      "Resource": "*"
    }
  ]
}
```

 Name it:

```
Week2-Security-Challenge-OverPrivileged
```

 Attach it to:

```
week2-developer
```

---

 # Phase 2: Student Challenge

 Give students the following scenario:

 > **You are the security analyst responsible for reviewing the `week2-developer` IAM identity. The identity was created by a developer and appears to have excessive permissions. Your job is to identify the security problems, determine the risk, reduce the permissions, and document your findings.**

---

 # Phase 3: Identify Current Permissions

 Students inspect:

 **IAM → Users → week2-developer → Permissions**

 They should document:

```
Identity:
week2-developer

Attached policies:
____________________

S3 permissions:
____________________

EC2 permissions:
____________________

Lambda permissions:
____________________

IAM permissions:
____________________
```

---

 # Phase 4: Identify Excessive Permissions

 Students create a table:

 | Permission | Required? | Risk | Recommendation |
| --- | --- | --- | --- |
| `s3:GetObject` | Yes | Low | Keep |
| `s3:ListBucket` | Yes | Low | Keep |
| `s3:DeleteObject` | No | High | Remove |
| `s3:*` | No | Critical | Remove |
| `ec2:DescribeInstances` | Yes | Low | Keep |
| `ec2:TerminateInstances` | No | High | Remove |
| `ec2:*` | No | Critical | Remove |
| `lambda:*` | No | High | Remove |

---

 # Phase 5: Build the Least-Privilege Replacement

 Students create:

```
Week2-Developer-Final
```

 with:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DeveloperS3Access",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::week2-iam-lab-2026-<unique-name>"
    },
    {
      "Sid": "DeveloperS3ObjectAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::week2-iam-lab-2026-<unique-name>/*"
    },
    {
      "Sid": "DeveloperEC2Inspection",
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

 Replace `<unique-name>` with the actual bucket name.

---

 # Phase 6: Validate the Remediation

 ## Test 1 — S3 Read

```
aws s3 ls \
s3://week2-iam-lab-2026-<unique-name> \
--profile week2-developer
```

 Expected:

```
SUCCESS
```

---

 ## Test 2 — S3 Upload

```
echo "Least privilege test" > least-privilege.txt
```

 Then:

```
aws s3 cp \
least-privilege.txt \
s3://week2-iam-lab-2026-<unique-name>/least-privilege.txt \
--profile week2-developer
```

 Expected:

```
upload: ./least-privilege.txt
```

---

 ## Test 3 — S3 Delete

```
aws s3api delete-object \
--bucket week2-iam-lab-2026-<unique-name> \
--key least-privilege.txt \
--profile week2-developer
```

 Expected:

```
AccessDenied
```

---

 ## Test 4 — EC2 Inspection

```
aws ec2 describe-instances \
--profile week2-developer \
--region us-east-1
```

 Expected:

```
SUCCESS
```

---

 ## Test 5 — EC2 Modification

 Use a nonexistent/fake instance ID:

```
aws ec2 terminate-instances \
--instance-ids i-00000000000000000 \
--profile week2-developer \
--region us-east-1
```

 Expected:

```
AccessDenied
```

---

 **"Can I access AWS?" → "Who am I?" → "What can I do?" → "Why can I do it?" → "Do I have too much access?" → "How do I reduce it?" → "Can I prove the identity is secure?"**
