````
# Phase 1: Set Up IAM and Resources via the AWS Console

## Step 1: Create a Custom Least-Privilege Policy

- Log in to the AWS Management Console using your administrator credentials and search for IAM.
- In the left navigation pane, choose **Policies → Create policy**.
- Select the **JSON** tab and paste the following snippet to scope access to a specific S3 bucket and EC2 inspection actions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3SpecificBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-demo-bucket-2026",
        "arn:aws:s3:::my-demo-bucket-2026/*"
      ]
    },
    {
      "Sid": "AllowEC2Describe",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeSecurityGroups"
      ],
      "Resource": "*"
    }
  ]
}
````

 - Click **Next**, name the policy `DemoResourcePolicy`, and click **Create policy**.

 ## Step 2: Create a User Group and Attach the Policy

 - In the IAM sidebar, click **User groups → Create group**.
- Name the group `DemoCLI-Group`.
- Under **Attach permissions policies**, search for and select `DemoResourcePolicy`.
- Click **Create group**.

 ## Step 3: Create the Demo User and Generate Access Keys

 - In the IAM sidebar, click **Users → Create user**.
- Name the user `demo-cli-operator`.
- Leave console access unchecked since this is a CLI-focused demo.
- Click **Next**.
- Select **Add user to group**, and choose `DemoCLI-Group`.
- Click **Next**, then **Create user**.
- Click on your newly created user `demo-cli-operator` from the list.
- Go to the **Security credentials** tab and scroll down to **Access keys**.
- Click **Create access key**.
- Select **Command Line Interface (CLI)**, acknowledge the recommendation, and click **Next**.
- Complete the wizard and copy/download your **Access Key ID** and **Secret Access Key**.

 ## Step 4: Create the Target S3 Bucket in the Console

 - Search for and open the **S3** service in the console.
- Click **Create bucket**.
- Name it `my-demo-bucket-2026` (ensure it matches the bucket name defined in your IAM policy).
- Keep **Block Public Access** enabled and click **Create bucket**.

 # Phase 2: Configure the Local AWS CLI Profile

 Open your local terminal and map the credentials you just generated to a dedicated CLI profile:

```
aws configure --profile demo-profile
```

 Enter the following when prompted:

 - **AWS Access Key ID:** Paste your generated Access Key ID.
- **AWS Secret Access Key:** Paste your generated Secret Access Key.
- **Default region name:** Enter your target region (e.g., `us-east-1`).
- **Default output format:** Enter `json`.

 # Phase 3: Demonstrate via the AWS CLI

 Now, use your terminal to demonstrate how your profile handles both permitted and restricted actions.

 ## 1\. Demonstrate Allowed S3 Access

 Run a command to check the target bucket contents:

```
aws s3 ls s3://my-demo-bucket-2026 --profile demo-profile
```

 > **Expected Output:** Success (lists empty or uploaded contents).

 ## 2\. Demonstrate Blocked/Unauthorized S3 Access

 Try listing a bucket that was not included in your IAM policy:

```
aws s3 ls s3://some-other-unauthorized-bucket --profile demo-profile
```

 > **Expected Output:** An error occurred (`AccessDenied`) when calling the `ListObjectsV2` operation... This proves that the least-privilege security boundaries are working.

 ## 3\. Demonstrate Allowed EC2 Visibility

 Query your EC2 instances to show view permissions:

```
aws ec2 describe-instances --profile demo-profile --region us-east-1
```

 > **Expected Output:** Success (returns a JSON list of instances or an empty `reservations` array, proving the `DescribeInstances` action succeeded).

 ## Optional: Test EC2 Write Restrictions

 Would you like to add an **EC2 launch or termination restriction test** to this workflow to show how write actions are prevented?

```

```
