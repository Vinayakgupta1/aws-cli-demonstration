# Week 3 — Core Cloud Services: Practical Demonstration Scripts

# Day 15 — EC2 Security

## Phase 1: Launch an EC2 Instance

### Step 1: Open EC2

- Log in to the AWS Management Console.
- Search for EC2.
- Open Instances → Launch instances.

### Step 2: Configure the Instance

Use the following configuration:

- Name: `week3-web-server`
- AMI: Amazon Linux 2023
- Instance type: A small eligible instance, such as `t3.micro`, subject to regional availability and pricing.
- Key pair: Create a new key pair named `week3-demo-key` and download the private key securely.
- Network: Select the default VPC for this introductory demonstration, or the custom VPC created later in the week.
- Auto-assign public IP: Enable it only if you need direct SSH access for this demo.
- Security group: Create `week3-ec2-sg`.

Configure the inbound rules as follows:

| Type | Port | Source |
| --- | --- | --- |
| SSH | 22 | Your current public IP `/32` |
| HTTP | 80 | `0.0.0.0/0`, only if demonstrating a public web server |

Do not open SSH to `0.0.0.0/0`.

- Keep the root EBS volume encrypted.
- Leave instance termination protection disabled for this disposable lab.
- Click Launch instance.

## Phase 2: Inspect EC2 Security

### Step 1: Check Instance Details

Open the instance and inspect:

- AMI ID
- Instance type
- Instance state
- Public IPv4 address
- Private IPv4 address
- VPC ID and subnet ID
- Attached EBS volumes
- Associated security groups

Expected result: The instance shows its configuration, network placement, and storage details.

### Step 2: Demonstrate Security Group Restrictions

- Open Security → Security groups.
- Select `week3-ec2-sg`.
- Inspect its inbound and outbound rules.
- Change the SSH source to a different test IP range that does not include your current IP.
- Attempt to reconnect using SSH.
- Restore the SSH rule to your current IP afterward.

Expected result: The SSH connection should fail or time out when the source IP is not permitted. This demonstrates network-level access control; it does not prove that the SSH key itself is invalid.

### Step 3: Demonstrate Instance Metadata Protection

For a safe metadata-security demonstration, inspect the instance's metadata configuration:

- Select the instance and choose Actions → Instance settings → Modify instance metadata options.
- Check whether IMDSv2 is required.
- If it is not required, enable IMDSv2 only after verifying application compatibility.

Expected result: The instance requires token-based metadata requests when configured for IMDSv2 only. Avoid displaying temporary credentials or other sensitive metadata during the demo.

## Phase 3: Demonstrate Using AWS CLI

Configure an existing AWS CLI profile with appropriate EC2 read permissions.

Check the instance:

bash
```
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=week3-web-server" \
  --query "Reservations[].Instances[].{ID:InstanceId,State:State.Name,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress,Type:InstanceType}" \
  --output table \
  --region us-east-1
```

Inspect the security group:

bash
```
aws ec2 describe-security-groups \
  --group-names week3-ec2-sg \
  --region us-east-1
```

Expected result: The commands return the instance details and configured security rules, provided the security group name is unique in the selected VPC and the CLI identity has permission to inspect them.

# Day 16 — S3 Security

## Phase 1: Create a Secure S3 Bucket

### Step 1: Create the Bucket

- Open Amazon S3 → Buckets → Create bucket.
- Enter a globally unique name, such as `week3-demo-<unique-suffix>`.
- Select `us-east-1`.
- Keep Block all public access enabled.
- Enable Bucket Versioning.
- Keep default server-side encryption enabled, preferably SSE-S3 for this introductory lab.
- Create the bucket.

### Step 2: Upload a Test Object

Create a local file named `demo.txt` containing:

text
```
This is a test file for demonstrating S3 security.
```

Upload it to the bucket using the console.

Expected result: The object appears in the bucket and is not publicly accessible.

## Phase 2: Demonstrate S3 Access Controls

### Step 1: Inspect Public Access Settings

- Open the bucket's Permissions tab.
- Inspect Block public access.
- Review the bucket policy and ACL settings.
- Confirm that public access is blocked.

### Step 2: Test Authorized Access Using the CLI

Set the bucket name in your terminal:

bash
```
BUCKET="week3-demo-your-unique-suffix"
```

List its contents:

bash
```
aws s3 ls "s3://$BUCKET" --region us-east-1
```

Read the test object:

bash
```
aws s3 cp "s3://$BUCKET/demo.txt" ./downloaded-demo.txt \
  --region us-east-1
```

Expected result: Both operations succeed when the configured identity has the necessary S3 permissions.

### Step 3: Demonstrate Denied Access

Use a separate test identity with no permissions for the bucket, or temporarily use an authorized test role that explicitly denies the required action.

bash
```
aws s3 ls "s3://$BUCKET" \
  --profile unauthorized-test \
  --region us-east-1
```

Expected result: `AccessDenied`, provided the test identity truly lacks access and no other applicable permissions grant it.

### Step 4: Demonstrate Public Access Protection

- Keep Block Public Access enabled.
- In a browser, attempt to open the object's URL without signing in.
- Observe the access-denied response.

Do not disable public access on a bucket containing real or sensitive data just to demonstrate exposure.

Expected result: Anonymous access is denied.

## Phase 3: Demonstrate Versioning

Upload a modified version of `demo.txt` with different contents.

Run:

bash
```
aws s3api list-object-versions \
  --bucket "$BUCKET" \
  --prefix demo.txt \
  --region us-east-1
```

Expected result: Multiple object versions appear, allowing you to demonstrate recovery from accidental overwrites or deletions.

# Day 17 — VPC Fundamentals

## Phase 1: Create a Custom VPC

### Step 1: Create the VPC

- Open VPC → Your VPCs → Create VPC.
- Choose VPC only.
- Enter:
  - Name: `week3-vpc`
  - IPv4 CIDR: `10.20.0.0/16`
- Create the VPC.

### Step 2: Create Two Subnets

Open Subnets → Create subnet and create these subnets in the VPC.

| Setting | Public subnet | Private subnet |
| --- | --- | --- |
| Name | `week3-public-subnet` | `week3-private-subnet` |
| IPv4 CIDR | `10.20.1.0/24` | `10.20.2.0/24` |
| Availability Zone | One AZ | Same AZ for this basic demo |

Expected result: Two subnets exist within `week3-vpc`.

## Phase 2: Configure Internet Routing

### Step 1: Create an Internet Gateway

- Open Internet gateways → Create internet gateway.
- Name it `week3-igw`.
- Attach it to `week3-vpc`.

### Step 2: Configure the Public Route Table

- Open Route tables → Create route table.
- Name it `week3-public-rt`.
- Associate it with `week3-public-subnet`.
- Add this route:

| Destination | Target |
| --- | --- |
| `10.20.0.0/16` | Local |
| `0.0.0.0/0` | `week3-igw` |

Expected result: The public subnet has a route to the internet gateway. An instance also needs a public IPv4 address and permissive-enough security rules to be reachable directly from the internet.

### Step 3: Configure the Private Route Table

- Create a route table named `week3-private-rt`.
- Associate it with `week3-private-subnet`.
- Keep the local route.

For this demonstration, do not add an internet route.

Expected result: The private subnet has no direct route to the internet through an internet gateway or NAT Gateway.

## Phase 3: Demonstrate Network Isolation

- Launch an EC2 instance into the public subnet.
- Launch a second instance into the private subnet, without a public IPv4 address.
- Configure security groups separately.
- Inspect both instances' subnet associations, IP addresses, and route tables.

Expected result:

- The public instance can potentially receive inbound connections when its public IP, routes, and security group allow them.
- The private instance has no direct internet route and cannot be reached directly from the internet.
- A private instance may still reach other resources inside the VPC, subject to routing and security controls.

Note: A NAT Gateway is needed when private IPv4 instances require outbound internet connectivity through a conventional NAT design. It is not required for this isolated routing demonstration, and it can incur significant charges.

# Day 18 — Security Groups vs NACLs

## Phase 1: Demonstrate Security Group Rules

### Step 1: Select an EC2 Instance

- Open EC2 → Instances.
- Select the demonstration instance.
- Open its attached security group.

### Step 2: Configure Inbound Rules

Use these example rules for a public web-server demonstration:

| Type | Port | Source |
| --- | --- | --- |
| HTTP | 80 | `0.0.0.0/0` |
| SSH | 22 | Your current public IP `/32` |

- Save the rules.
- Test HTTP access if a web server is installed and running.
- Test SSH access from an authorized source.

Expected result: Traffic matching the allowed inbound rules can proceed, subject to other network and host controls. Security groups are stateful, so response traffic for permitted connections is automatically allowed.

## Phase 2: Demonstrate Network ACLs

### Step 1: Open the Subnet's Network ACL

- Open VPC → Network ACLs.
- Identify the ACL associated with the demonstration subnet.
- Inspect inbound and outbound rules.

### Step 2: Add a Test Rule

For an isolated test subnet, add a narrowly scoped inbound rule to deny a selected test port or source IP. Make sure the rule number takes precedence over conflicting rules.

For example, you can deny inbound TCP port `8080` using an appropriate rule number and source CIDR.

Expected result: Matching traffic is denied at the subnet boundary, even if the instance security group allows it.

Restore the original rules after the demonstration.

## Phase 3: Compare Both Controls

| Feature | Security Group | Network ACL |
| --- | --- | --- |
| Scope | Network interface/resource | Subnet |
| State | Stateful | Stateless |
| Rules | Allow rules | Allow and deny rules |
| Return traffic | Automatically allowed for established permitted flows | Must be permitted by applicable rules |
| Rule evaluation | All applicable rules are evaluated | Rules evaluated in ascending rule-number order; first match applies |

Demonstration conclusion: Security groups provide the primary resource-level traffic control, while NACLs provide an additional subnet-level filter. Both must be considered when troubleshooting connectivity.

# Day 19 — RDS Security

## Phase 1: Create a Database Network Configuration

### Step 1: Create a DB Subnet Group

- Open Amazon RDS → Subnet groups → Create DB subnet group.
- Name it `week3-db-subnet-group`.
- Select `week3-vpc`.
- Add subnets from at least two Availability Zones for a typical production-oriented DB subnet group.

For the demonstration, use private subnets. If you only created one private subnet on Day 17, create another private subnet in a different Availability Zone first.

### Step 2: Configure the Database Security Group

- Open EC2 → Security groups → Create security group.
- Name it `week3-rds-sg`.
- Select `week3-vpc`.
- Add this inbound rule:

| Type | Port | Source |
| --- | --- | --- |
| MySQL/Aurora | 3306 | The EC2 application's security group |

Do not use `0.0.0.0/0` as the database source.

## Phase 2: Create the RDS Database

### Step 1: Launch RDS

- Open RDS → Databases → Create database.
- Select Standard create.
- Choose MySQL.
- Use a suitable development/test template.
- Set the DB identifier to `week3-db`.
- Configure the master username and a strong password.
- Select the VPC `week3-vpc`.
- Select `week3-db-subnet-group`.
- Set Public access to No.
- Attach `week3-rds-sg`.
- Enable storage encryption.
- Configure backups and deletion protection according to the demonstration's cleanup requirements.
- Create the database.

Expected result: RDS provisions a database endpoint inside the selected VPC. Provisioning may take several minutes, and database instances incur charges while running.

## Phase 3: Connect EC2 to RDS

### Step 1: Prepare the EC2 Application Server

Use an EC2 instance in the same VPC with network routes to the database subnets. Ensure the EC2 security group is the source allowed by `week3-rds-sg`.

Install a MySQL client if one is not already available.

### Step 2: Test Database Connectivity

Obtain the database endpoint from the RDS console.

bash
```
mysql -h YOUR_RDS_ENDPOINT \
  -P 3306 \
  -u YOUR_DB_USERNAME \
  -p
```

Enter the password at the prompt rather than including it in the command.

After connecting, run:

sql
```
SELECT VERSION();
SHOW DATABASES;
```

Expected result: The client connects if routing, security groups, credentials, and database availability are correctly configured.

### Step 3: Demonstrate Database Isolation

- Attempt to connect from an unauthorized EC2 instance or test security group.
- Confirm that the source is not allowed by `week3-rds-sg`.
- Test the connection again from the authorized EC2 instance.

Expected result: The unauthorized network connection should fail, while the permitted connection can succeed.

Security point: A private database should not be directly exposed to the internet. In a production design, use managed secrets, restricted database accounts, TLS, and appropriate backup and recovery controls.

# Day 20 — Lambda and API Gateway

## Phase 1: Create a Lambda Function

### Step 1: Create the Function

- Open AWS Lambda → Create function.
- Select Author from scratch.
- Name the function `week3-demo-function`.
- Choose a supported Python runtime.
- Create or select an execution role with only the permissions the function requires.
- Create the function.

### Step 2: Add a Test Function

Replace the default function code with:

python
```
import json

def lambda_handler(event, context):
    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json"
        },
        "body": json.dumps({
            "message": "Hello from AWS Lambda",
            "service": "week3-demo"
        })
    }
```

Deploy the code.

### Step 3: Test the Function

- Open the Test tab.
- Create a basic test event.
- Run the test.

Expected result: Lambda returns a successful response with a JSON message.

## Phase 2: Expose Lambda Through API Gateway

### Step 1: Create an HTTP API

- Open Amazon API Gateway.
- Choose Create API → HTTP API.
- Add an integration with `week3-demo-function`.
- Configure a route such as `GET /hello`.
- Create a stage, using the default stage if appropriate.
- Deploy the API.

### Step 2: Test the Endpoint

Copy the invoke URL shown by API Gateway. Append the configured route if the URL does not already include it.

Run:

bash
```
curl -i "YOUR_API_INVOKE_URL/hello"
```

Expected result: The API returns HTTP 200 and the Lambda JSON response.

## Phase 3: Demonstrate API Security

### Step 1: Inspect API Exposure

- Open the API Gateway configuration.
- Review the invoke URL, routes, stages, and authorization settings.
- Confirm whether the endpoint is publicly reachable.

An HTTP API endpoint is generally reachable over the internet unless its design and access controls restrict it. A publicly reachable endpoint is not automatically an authorized endpoint.

### Step 2: Compare Access-Control Options

For an introductory demonstration, discuss or configure one of these approaches:

- IAM authorization: Requests must be signed with AWS credentials authorized to invoke the API.
- JWT authorization: Requests must carry a valid token from a configured identity provider.
- Lambda authorizer: A Lambda function evaluates whether a request is authorized.
- Throttling and monitoring: Limit excessive requests and monitor API activity.

Avoid placing AWS credentials or secret keys inside client-side JavaScript or mobile applications.

Expected result: The configured authorization mechanism determines whether a request is accepted or rejected. Test both an authorized request and an unauthorized request when authentication is enabled.

# Day 21 — Architecture Challenge: Two-Tier Application with S3

## Phase 1: Build the Architecture

The final demonstration combines EC2, RDS, VPC, security groups, and S3 into one cloud environment.

Internet / User

HTTP/HTTPS requests

EC2 — Application Tier

Public subnet · Web application

Port 3306 · MySQL

RDS — Database Tier

Private subnet · No public access

S3 — Object Storage

Private bucket · Controlled application access

S3 is a separate storage service; it is not directly connected to RDS. The application accesses S3 through AWS APIs using its assigned IAM permissions.

## Phase 2: Configure the Network

### Step 1: Verify the VPC

Open the VPC console and confirm:

- VPC: `week3-vpc`
- Public subnet: `week3-public-subnet`
- Private database subnets in at least two Availability Zones
- Internet Gateway attached to the VPC
- Public route table with a route to the Internet Gateway
- Private database subnets without direct internet routes

### Step 2: Configure the Application Security Group

Create or update `week3-app-sg`.

| Type | Port | Source |
| --- | --- | --- |
| HTTP | 80 | `0.0.0.0/0`, if demonstrating public HTTP |
| HTTPS | 443 | `0.0.0.0/0`, if HTTPS is configured |
| SSH | 22 | Your public IP `/32`, only if required |

For a production deployment, prefer HTTPS and managed administrative access such as AWS Systems Manager instead of exposing SSH.

### Step 3: Configure the Database Security Group

Set the inbound rule on `week3-rds-sg` to:

| Type | Port | Source |
| --- | --- | --- |
| MySQL/Aurora | 3306 | `week3-app-sg` |

Do not allow inbound database connections from the entire internet.

## Phase 3: Connect the Application to RDS

- Open the RDS console and copy the database endpoint.
- Verify that the application EC2 instance can route to the database.
- Configure the application with the database endpoint, database name, username, and password.
- Store database credentials in AWS Secrets Manager or another appropriate secret-management solution rather than hardcoding them into application code.
- Test a database connection from the application instance.

Expected result: The application can connect to RDS, while other network sources remain blocked unless explicitly permitted.

## Phase 4: Connect the Application to S3

### Step 1: Create the Application Bucket

- Open Amazon S3.
- Create a bucket named `week3-app-storage-<unique-suffix>`.
- Keep Block Public Access enabled.
- Enable versioning.
- Keep encryption enabled.

### Step 2: Assign an IAM Role to EC2

- Open IAM and create an EC2 role named `week3-app-role`.
- Trust the EC2 service.
- Attach a least-privilege policy scoped to the application bucket.

Example policy for listing the bucket and reading or uploading objects:

json
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListApplicationBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::week3-app-storage-UNIQUE_SUFFIX"
    },
    {
      "Sid": "ReadWriteApplicationObjects",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::week3-app-storage-UNIQUE_SUFFIX/*"
    }
  ]
}
```

Replace `UNIQUE_SUFFIX` in both resource ARNs with the actual bucket suffix. Attach the role to the EC2 instance through EC2 → Actions → Security → Modify IAM role.

### Step 3: Test S3 Access from EC2

On the EC2 instance, create a test file:

bash
```
echo "Week 3 architecture demonstration" > demo.txt
```

Upload the object:

bash
```
aws s3 cp demo.txt \
  s3://YOUR_APPLICATION_BUCKET/demo.txt \
  --region us-east-1
```

List the bucket:

bash
```
aws s3 ls s3://YOUR_APPLICATION_BUCKET/ \
  --region us-east-1
```

Expected result: The application instance can upload and list objects using its IAM role, without storing long-lived AWS access keys on the instance.
