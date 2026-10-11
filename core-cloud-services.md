WEEK 3 — CORE CLOUD SERVICES
DAY 15 — EC2 SECURITY
Phase 1: Launch an EC2 Instance via the AWS Console
Step 1: Open EC2
Log in to the AWS Management Console using your administrator/training credentials.
Search for EC2.
Open Instances → Launch instances.
Step 2: Configure the EC2 Instance

Configure the instance as follows:

Setting	Value
Name	week3-web-server
AMI	Amazon Linux 2023
Instance type	t3.micro, if eligible and available
Key pair	week3-demo-key
VPC	Default VPC for the initial demonstration
Auto-assign public IP	Enable only if required for direct SSH access
Security group	week3-ec2-sg
Create a new key pair named:
week3-demo-key

Download and securely store the private key.
Create the security group:
week3-ec2-sg


Configure the inbound rules:

Type	Port	Source
SSH	22	Your current public IP /32
HTTP	80	0.0.0.0/0, only for a public web-server demonstration
Do not allow SSH access from 0.0.0.0/0.
Keep the root EBS volume encrypted.
Leave termination protection disabled for this disposable lab.
Click Launch instance.
Expected Result

The EC2 instance launches with the selected AMI, instance type, network settings, and security group.

Phase 2: Inspect EC2 Security
Step 1: Check Instance Details
Open EC2 → Instances.
Select week3-web-server.
Inspect the following:
AMI ID
Instance type
Instance state
Public IPv4 address
Private IPv4 address
VPC ID
Subnet ID
Attached EBS volumes
Associated security groups
Expected Result

The instance displays its configuration, network placement, and storage details.

Step 2: Demonstrate Security Group Restrictions
Open EC2 → Security groups.
Select week3-ec2-sg.
Inspect inbound and outbound rules.
Temporarily change the SSH source to a test IP range that does not include your current public IP.
Attempt to reconnect using SSH.
Restore the original SSH rule afterward.
Expected Result

The SSH connection should fail or time out when the connecting IP is not permitted.

This demonstrates network-level access control, not whether the SSH key is valid.

Step 3: Demonstrate Instance Metadata Protection
Open EC2 → Instances.
Select the instance.
Choose Actions → Instance settings → Modify instance metadata options.
Check whether IMDSv2 is required.
If IMDSv2 is not required, enable it after verifying application compatibility.
Expected Result

When the instance is configured to require IMDSv2, metadata requests must use session tokens.

Do not display temporary credentials or sensitive instance metadata during the demonstration.

Phase 3: Demonstrate EC2 Using AWS CLI
Step 1: Configure the AWS CLI

Use an existing AWS CLI profile with the required EC2 read permissions.

Verify your AWS identity:

aws sts get-caller-identity \
  --region us-east-1

Step 2: Inspect the EC2 Instance

Run:

aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=week3-web-server" \
  --query "Reservations[].Instances[].{ID:InstanceId,State:State.Name,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress,Type:InstanceType}" \
  --output table \
  --region us-east-1

Expected Output

A table containing the instance ID, state, private IP, public IP, and instance type.

The result may be empty if the instance has not been created in us-east-1.

Step 3: Inspect the Security Group

Run:

aws ec2 describe-security-groups \
  --group-names week3-ec2-sg \
  --region us-east-1

Expected Result

The command displays the security group configuration if the group exists in the selected region and the CLI identity has permission to inspect it.

DAY 16 — S3 SECURITY
Phase 1: Create the S3 Bucket via the AWS Console
Step 1: Open Amazon S3
Log in to the AWS Management Console.
Search for S3.
Open Buckets → Create bucket.
Step 2: Configure the Bucket

Enter the following bucket name:

week3-core-cloud-lab-2026-vinayak


Configure the bucket:

Setting	Value
Bucket name	week3-core-cloud-lab-2026-vinayak
AWS Region	us-east-1
Block all public access	Enabled
Bucket Versioning	Enabled
Default encryption	Enabled, preferably SSE-S3
Keep Block Public Access enabled.
Enable Bucket Versioning.
Keep default server-side encryption enabled.
Click Create bucket.
S3 bucket names must be globally unique across AWS accounts. If this name is already taken, select another globally unique name and update all commands and policy ARNs accordingly.
Phase 2: Upload a Test Object
Step 1: Create a Test File

On your local machine, create a file named demo.txt.

Run:

echo "This is a test file for demonstrating S3 security." > demo.txt

Step 2: Upload the File
Open the bucket week3-core-cloud-lab-2026-vinayak.
Click Upload.
Add demo.txt.
Click Upload.
Expected Result

The object appears in the bucket and is not publicly accessible.

Phase 3: Demonstrate S3 Access Controls Using AWS CLI
Step 1: Set the Bucket Name

Open your terminal and run:

BUCKET="week3-core-cloud-lab-2026-vinayak"

Step 2: List Bucket Contents

Run:

aws s3 ls "s3://$BUCKET" \
  --region us-east-1

Expected Result

The command lists the objects in the bucket if your AWS identity has the required permissions.

Step 3: Download the Test Object

Run:

aws s3 cp "s3://$BUCKET/demo.txt" ./downloaded-demo.txt \
  --region us-east-1

Expected Output
download: s3://week3-core-cloud-lab-2026-vinayak/demo.txt to ./downloaded-demo.txt


The exact output formatting may vary.

Step 4: Demonstrate Denied Access

Use a separate test identity that does not have permission to access the bucket.

Run:

aws s3 ls "s3://$BUCKET" \
  --profile unauthorized-test \
  --region us-east-1

Expected Output
AccessDenied


This result depends on the test identity genuinely lacking access and not receiving permission through another applicable policy.

Step 5: Demonstrate Public Access Protection
Open the bucket's Permissions tab.
Verify that Block Public Access is enabled.
Copy the object URL.
Open the URL in a browser without signing in.
Expected Result

Anonymous access to the private object is denied.

Do not disable public access on a bucket containing real or sensitive data to demonstrate exposure.

Phase 4: Demonstrate S3 Versioning
Step 1: Modify the Test File

Change the file contents:

echo "Updated version of the S3 security demonstration." > demo.txt


Upload the modified file:

aws s3 cp demo.txt "s3://$BUCKET/demo.txt" \
  --region us-east-1

Step 2: List Object Versions

Run:

aws s3api list-object-versions \
  --bucket "$BUCKET" \
  --prefix demo.txt \
  --region us-east-1

Expected Result

The output shows the available object versions and their version IDs.

This demonstrates how S3 versioning can help recover from accidental overwrites or deletions.

DAY 17 — VPC FUNDAMENTALS
Phase 1: Create a Custom VPC
Step 1: Open the VPC Console
Open the AWS Management Console.
Search for VPC.
Select Your VPCs → Create VPC.
Choose VPC only.

Enter:

Name: week3-vpc
IPv4 CIDR: 10.20.0.0/16

Click Create VPC.
Step 2: Create the Public Subnet
Open VPC → Subnets → Create subnet.
Select week3-vpc.
Enter:
Subnet name: week3-public-subnet
IPv4 CIDR: 10.20.1.0/24

Select an Availability Zone.
Create the subnet.
Step 3: Create the Private Subnet

Create another subnet in the same VPC.

Enter:

Subnet name: week3-private-subnet
IPv4 CIDR: 10.20.2.0/24


For this introductory demonstration, you can use the same Availability Zone as the public subnet.

Expected Result

The VPC contains two subnets:

week3-vpc
    |
    ├── week3-public-subnet
    |      10.20.1.0/24
    |
    └── week3-private-subnet
           10.20.2.0/24

Phase 2: Configure Internet Routing
Step 1: Create an Internet Gateway
Open VPC → Internet gateways.
Click Create internet gateway.
Enter:
week3-igw

Create the gateway.
Select it and choose Actions → Attach to a VPC.
Attach it to week3-vpc.
Step 2: Create the Public Route Table
Open VPC → Route tables → Create route table.
Enter:
Name: week3-public-rt
VPC: week3-vpc

Create the route table.
Open Subnet associations → Edit subnet associations.
Associate week3-public-subnet.

Add the following route:

Destination	Target
10.20.0.0/16	Local
0.0.0.0/0	week3-igw
Expected Result

The public subnet has a route to the internet gateway.

An EC2 instance also needs a public IPv4 address and appropriate security group rules to be directly reachable from the internet.

Step 3: Create the Private Route Table
Open VPC → Route tables → Create route table.
Enter:
Name: week3-private-rt
VPC: week3-vpc

Create the route table.
Associate it with week3-private-subnet.
Keep the local route.
Do not add an internet gateway or NAT Gateway route for this demonstration.
Expected Result

The private subnet has no direct route to the internet.

Phase 3: Demonstrate Network Isolation
Step 1: Launch an EC2 Instance in the Public Subnet
Open EC2 → Launch instances.
Select Amazon Linux 2023.
Select week3-vpc.
Select week3-public-subnet.
Enable auto-assignment of a public IP if required.
Attach an appropriate security group.
Launch the instance.
Step 2: Launch an EC2 Instance in the Private Subnet
Launch a second EC2 instance.
Select week3-vpc.
Select week3-private-subnet.
Disable public IPv4 assignment.
Attach an appropriate security group.
Launch the instance.
Step 3: Inspect Network Placement

Verify the instances':

VPC IDs
Subnet IDs
Private IP addresses
Public IP addresses
Security groups
Associated route tables
Expected Architecture
                 Internet
                    |
                    v
              Internet Gateway
                    |
                    v
             Public Route Table
                    |
                    v
             Public Subnet
                    |
                    v
             Public EC2 Instance

                 week3-vpc
                    |
                    v
             Private Route Table
                    |
                    v
             Private Subnet
                    |
                    v
             Private EC2 Instance

Expected Result
The public instance can potentially receive inbound connections when its public IP, routes, and security group allow them.
The private instance cannot be reached directly from the internet.
Resources inside the VPC may still communicate when routing and security rules permit.
A NAT Gateway is needed if private IPv4 instances require outbound internet access through a conventional NAT design. It is not required for this isolated routing demonstration and can incur significant charges.
DAY 18 — SECURITY GROUPS VS NACLs
Phase 1: Demonstrate Security Group Rules
Step 1: Open the EC2 Security Group
Open EC2 → Instances.
Select the demonstration instance.
Open the Security tab.
Select the attached security group.

For the final architecture, you can use week3-app-sg. For the initial EC2 demonstration, you can use week3-ec2-sg.

Step 2: Configure Inbound Rules

Use these example rules for a public web-server demonstration:

Type	Port	Source
HTTP	80	0.0.0.0/0
SSH	22	Your current public IP /32
Save the rules.
Test HTTP access if a web server is installed and running.
Test SSH access from an authorized source.
Expected Result

Traffic matching allowed inbound rules can proceed, subject to other network and host controls.

Security groups are stateful, so response traffic for permitted connections is automatically allowed.

Phase 2: Demonstrate Network ACLs
Step 1: Open the Network ACL
Open VPC → Network ACLs.
Identify the network ACL associated with the demonstration subnet.
Inspect inbound and outbound rules.
Step 2: Configure a Test Rule

For an isolated test subnet:

Add a narrowly scoped inbound deny rule for a selected test port or source IP.
Use a rule number that takes precedence over conflicting rules.
For example, configure a deny rule for inbound TCP port 8080 using an appropriate rule number and source CIDR.
Expected Result

Matching traffic is denied at the subnet boundary even if the instance security group allows it.

Restore the original rules after the demonstration.

Phase 3: Compare Security Groups and NACLs
Feature	Security Group	Network ACL
Scope	Network interface/resource	Subnet
State	Stateful	Stateless
Rules	Allow rules	Allow and deny rules
Return traffic	Automatically allowed for permitted established flows	Must be permitted by applicable rules
Rule evaluation	All applicable rules are evaluated	Rules evaluated in ascending rule-number order; first match applies
Demonstration Conclusion
Security Group
      |
      v
Resource-Level Traffic Control

Network ACL
      |
      v
Subnet-Level Traffic Control


Security groups provide resource-level traffic control, while NACLs provide an additional subnet-level filter.

Both controls must be considered when troubleshooting network connectivity.

DAY 19 — RDS SECURITY
Phase 1: Create a Database Network Configuration
Step 1: Create a DB Subnet Group
Open Amazon RDS → Subnet groups.
Click Create DB subnet group.
Enter:
Name: week3-db-subnet-group
VPC: week3-vpc

Select private subnets in at least two Availability Zones for a typical production-oriented DB subnet group.
Create the subnet group.

If you created only one private subnet on Day 17, create another private subnet in a different Availability Zone before proceeding.

Step 2: Create the Database Security Group
Open EC2 → Security groups → Create security group.
Enter:
Security group name: week3-rds-sg
VPC: week3-vpc


Configure the inbound rule:

Type	Port	Source
MySQL/Aurora	3306	EC2 application's security group

For the final architecture, use week3-app-sg as the source.

For the initial demonstration, use the security group attached to the authorized EC2 application server.

Do not allow database connections from 0.0.0.0/0.

Phase 2: Create the RDS Database
Step 1: Open Amazon RDS
Open Amazon RDS → Databases → Create database.
Select Standard create.
Choose MySQL.
Select a suitable development/test template.

Configure:

DB identifier: week3-db
VPC: week3-vpc
DB subnet group: week3-db-subnet-group
Public access: No
Security group: week3-rds-sg
Storage encryption: Enabled

Configure a master username and strong password.
Configure backups and deletion protection according to your cleanup requirements.
Create the database.
Expected Result

RDS provisions a database endpoint inside the selected VPC.

Provisioning may take several minutes. RDS database instances incur charges while running.

Phase 3: Connect EC2 to RDS
Step 1: Prepare the EC2 Application Server
Use an EC2 instance in week3-vpc.
Ensure that the instance has network routes to the database subnets.
Verify that its security group is allowed by week3-rds-sg.
Install a MySQL client if one is not already available.
Step 2: Obtain the Database Endpoint
Open RDS → Databases → week3-db.
Locate the connectivity details.
Copy the database endpoint.
Step 3: Test Database Connectivity

Run the following command from the authorized EC2 instance:

mysql -h YOUR_RDS_ENDPOINT \
  -P 3306 \
  -u YOUR_DB_USERNAME \
  -p


Replace YOUR_RDS_ENDPOINT and YOUR_DB_USERNAME with your actual values.

Enter the password at the prompt instead of placing it directly in the command.

After connecting, run:

SELECT VERSION();
SHOW DATABASES;

Expected Result

The client connects if the routing, security groups, credentials, and database availability are correctly configured.

Phase 4: Demonstrate Database Isolation
Step 1: Test an Unauthorized Connection
Attempt to connect from an unauthorized EC2 instance or a test security group.
Verify that the source is not permitted by week3-rds-sg.
Expected Result

The unauthorized network connection should fail.

Step 2: Test an Authorized Connection
Connect from the authorized EC2 application server.
Use the correct database endpoint and credentials.
Expected Result

The permitted connection can succeed.

Security Architecture
EC2 Application
      |
      | TCP 3306
      v
week3-rds-sg
      |
      v
RDS MySQL
Private Subnets
No Public Access


A private database should not be directly exposed to the internet.

For production environments, use managed secrets, restricted database accounts, TLS, and appropriate backup and recovery controls.

DAY 20 — LAMBDA AND API GATEWAY
Phase 1: Create a Lambda Function
Step 1: Open AWS Lambda
Open the AWS Management Console.
Search for Lambda.
Click Create function.
Select Author from scratch.

Enter:

Function name: week3-demo-function
Runtime: A supported Python runtime

Create or select an execution role with only the permissions required by the function.
Click Create function.
Step 2: Add the Lambda Function Code

Replace the default function code with:

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

Click Deploy.
Step 3: Test the Lambda Function
Open the Test tab.
Create a basic test event.
Run the test.
Expected Output
{
  "statusCode": 200,
  "headers": {
    "Content-Type": "application/json"
  },
  "body": "{\"message\": \"Hello from AWS Lambda\", \"service\": \"week3-demo\"}"
}


The exact display format may vary in the console.

This demonstrates that the Lambda function executes and returns a JSON response.

Phase 2: Expose Lambda Through API Gateway
Step 1: Create an HTTP API
Open Amazon API Gateway.
Choose Create API → HTTP API.
Add an integration with week3-demo-function.
Configure the route:
GET /hello

Create a stage, using the default stage if appropriate.
Deploy the API.

Ensure that API Gateway has permission to invoke the Lambda function.

Step 2: Test the API Endpoint

Copy the invoke URL from API Gateway.

Run:

curl -i "YOUR_API_INVOKE_URL/hello"


Replace YOUR_API_INVOKE_URL with the actual invoke URL.

Expected Output

An HTTP 200 response containing the JSON message:

{
  "message": "Hello from AWS Lambda",
  "service": "week3-demo"
}

Phase 3: Demonstrate API Security
Step 1: Inspect API Exposure
Open the API Gateway configuration.
Review the invoke URL, routes, stages, and authorization settings.
Determine whether the endpoint is publicly reachable.

An HTTP API endpoint is generally reachable over the internet unless its design and access controls restrict it.

A publicly reachable endpoint is not automatically an authorized endpoint.

Step 2: Compare Access-Control Options

For the demonstration, discuss or configure one of the following approaches:

Security mechanism	Purpose
IAM authorization	Requires signed requests from authorized AWS identities
JWT authorization	Requires a valid token from a configured identity provider
Lambda authorizer	Uses a Lambda function to evaluate request authorization
Throttling and monitoring	Helps control excessive requests and monitor activity
Avoid placing AWS credentials or secret keys in client-side JavaScript or mobile applications.
Test both authorized and unauthorized requests when authentication is enabled.
Expected Result

The configured authorization mechanism determines whether a request is accepted or rejected.

DAY 21 — ARCHITECTURE CHALLENGE: TWO-TIER APPLICATION WITH S3
Phase 1: Build the Architecture

The final demonstration combines EC2, RDS, VPC, security groups, and S3 into one cloud environment.

Step 1: Understand the Architecture
                Internet / User
                      |
                 HTTP / HTTPS
                      |
                      v
             EC2 Application Tier
                Public Subnet
                      |
                 MySQL : 3306
                      |
                      v
               RDS Database Tier
                Private Subnets


             EC2 Application Tier
                      |
                AWS API / IAM Role
                      |
                      v
                S3 Object Storage
                Private S3 Bucket


S3 is a separate storage service and is not directly connected to RDS.

The application accesses S3 through AWS APIs using its assigned IAM role.

Phase 2: Configure the Network
Step 1: Verify the VPC

Open the VPC console and confirm the following resources:

VPC: week3-vpc

Public subnet:
week3-public-subnet

Private subnet:
week3-private-subnet

Internet Gateway:
week3-igw

Public Route Table:
week3-public-rt

Private Route Table:
week3-private-rt

Confirm the public route table has a route to the Internet Gateway.
Confirm the private database subnets do not have direct internet routes.
Ensure the RDS DB subnet group covers at least two Availability Zones.
Step 2: Configure the Application Security Group

Create or update the security group:

week3-app-sg


Associate it with the application EC2 instance in week3-vpc.

Configure the inbound rules:

Type	Port	Source
HTTP	80	0.0.0.0/0, if demonstrating public HTTP
HTTPS	443	0.0.0.0/0, if HTTPS is configured
SSH	22	Your public IP /32, only if required
Save the rules.
Avoid opening SSH to the entire internet.

For production deployments, prefer HTTPS and managed administrative access such as AWS Systems Manager.

Step 3: Configure the Database Security Group

Open week3-rds-sg and configure this inbound rule:

Type	Port	Source
MySQL/Aurora	3306	week3-app-sg

This permits database connections from the application security group.

Do not allow inbound database connections from the entire internet.

Phase 3: Connect the Application to RDS
Step 1: Obtain the Database Endpoint
Open Amazon RDS → Databases → week3-db.
Copy the database endpoint.
Verify that the EC2 application instance can route to the database.
Step 2: Configure Application Credentials

Configure the application with:

Database endpoint: YOUR_RDS_ENDPOINT
Database name: YOUR_DB_NAME
Database username: YOUR_DB_USERNAME
Database password: Retrieve securely


Store database credentials in AWS Secrets Manager or another suitable secret-management solution instead of hardcoding them into application code.

Step 3: Test Database Connectivity

From the application EC2 instance, run:

mysql -h YOUR_RDS_ENDPOINT \
  -P 3306 \
  -u YOUR_DB_USERNAME \
  -p


Enter the password when prompted.

Expected Result

The application can connect to RDS when routing, security groups, credentials, and database availability are correctly configured.

Other network sources remain blocked unless explicitly permitted.

Phase 4: Connect the Application to S3
Step 1: Create the Application Bucket
Open Amazon S3 → Buckets → Create bucket.
Enter this exact bucket name:
week3-core-cloud-lab-2026-vinayak


Configure:

Setting	Value
Bucket name	week3-core-cloud-lab-2026-vinayak
Region	us-east-1
Block Public Access	Enabled
Bucket Versioning	Enabled
Default encryption	Enabled
Create the bucket.
If the bucket name is already taken, choose a globally unique replacement and use it consistently throughout the lab.
Step 2: Create an IAM Role for EC2
Open IAM → Roles → Create role.
Select AWS service.
Select EC2.
Configure the trust relationship for the EC2 service.

Enter the role name:

week3-app-role

Create the role.
Attach a least-privilege permissions policy scoped to the application bucket.
Step 3: Create the S3 IAM Policy
Open IAM → Policies → Create policy.
Select the JSON tab.
Paste the following policy:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListApplicationBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::week3-core-cloud-lab-2026-vinayak"
    },
    {
      "Sid": "ReadWriteApplicationObjects",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::week3-core-cloud-lab-2026-vinayak/*"
    }
  ]
}

Click Next.
Enter the policy name:
Week3-App-S3-Policy

Create the policy.
Attach Week3-App-S3-Policy to week3-app-role.

The policy permits:

s3:ListBucket
s3:GetObject
s3:PutObject


It does not grant permission to delete objects or modify bucket settings.

Step 4: Attach the IAM Role to EC2
Open EC2 → Instances.
Select the application instance.
Choose Actions → Security → Modify IAM role.
Select:
week3-app-role

Save the change.
Expected Architecture
EC2 Application
       |
       v
week3-app-role
       |
       v
Week3-App-S3-Policy
       |
       ├── s3:ListBucket
       ├── s3:GetObject
       └── s3:PutObject
       |
       v
week3-core-cloud-lab-2026-vinayak


The EC2 instance obtains temporary credentials through its attached IAM role. Do not store long-lived AWS access keys on the instance.

Phase 5: Test S3 Access from EC2
Step 1: Create a Test File

Connect to the application EC2 instance.

Run:

echo "Week 3 architecture demonstration" > demo.txt

Step 2: Set the Bucket Name

Run:

BUCKET="week3-core-cloud-lab-2026-vinayak"

Step 3: Upload the Object

Run:

aws s3 cp demo.txt \
  "s3://$BUCKET/demo.txt" \
  --region us-east-1

Expected Output
upload: ./demo.txt to s3://week3-core-cloud-lab-2026-vinayak/demo.txt


The exact output formatting may vary.

Step 4: List Bucket Contents

Run:

aws s3 ls "s3://$BUCKET/" \
  --region us-east-1

Expected Result

The output lists demo.txt if the IAM role and bucket configuration permit the operation.

Step 5: Download the Object

Run:

aws s3 cp "s3://$BUCKET/demo.txt" ./downloaded-demo.txt \
  --region us-east-1

Expected Result

The object downloads successfully if the EC2 role has s3:GetObject permission.

Phase 6: Verify the Complete Architecture
Step 1: Verify EC2

Confirm:

Application instance:
week3-web-server

Application security group:
week3-app-sg


The application instance should use the correct VPC and subnet.

Step 2: Verify RDS

Confirm:

Database:
week3-db

Database security group:
week3-rds-sg

Database port:
3306

Public access:
No


Verify that only the intended application security group is allowed to initiate database connections.

Step 3: Verify S3

Confirm:

S3 bucket:
week3-core-cloud-lab-2026-vinayak

Block Public Access:
Enabled

Versioning:
Enabled

Encryption:
Enabled


Verify that the EC2 role can list, upload, and download objects.

Step 4: Review the Final Architecture
                    Internet
                       |
                       v
                Application EC2
                week3-app-sg
                       |
             +---------+---------+
             |                   |
             v                   v
        Private RDS          Private S3
        week3-db              Bucket
        Port 3306             Controlled
             ^                 IAM access
             |                   ^
             |                   |
             +---------+---------+
                       |
                 week3-app-role
                       |
                Week3-App-S3-Policy


The application communicates with RDS through VPC networking and security groups, and accesses S3 through AWS APIs using its IAM role.

FINAL WEEK 3 CHECKLIST
Day 15 — EC2 Security
Launch an EC2 instance.
Configure week3-ec2-sg.
Inspect instance details.
Demonstrate SSH restrictions.
Inspect IMDSv2 settings.
Run EC2 AWS CLI commands.
Day 16 — S3 Security
Create week3-core-cloud-lab-2026-vinayak.
Enable Block Public Access.
Enable versioning and encryption.
Upload demo.txt.
Test authorized and unauthorized access.
Demonstrate object versioning.
Day 17 — VPC Fundamentals
Create week3-vpc.
Create public and private subnets.
Attach week3-igw.
Configure public and private route tables.
Demonstrate network isolation.
Day 18 — Security Groups vs NACLs
Configure security group rules.
Inspect network ACL rules.
Demonstrate subnet-level filtering.
Compare stateful and stateless controls.
Day 19 — RDS Security
Create week3-db-subnet-group.
Create week3-rds-sg.
Launch the private MySQL database week3-db.
Connect from the authorized EC2 instance.
Demonstrate database isolation.
Day 20 — Lambda and API Gateway
Create week3-demo-function.
Deploy and test the Lambda code.
Create an API Gateway HTTP API.
Configure the GET /hello route.
Test the endpoint.
Review API authorization options.
Day 21 — Architecture Challenge
Verify the VPC and subnet configuration.
Configure week3-app-sg and week3-rds-sg.
Connect the application to RDS.
Create or verify the S3 bucket.
Create week3-app-role.
Attach Week3-App-S3-Policy.
Upload, list, and download an S3 object from EC2.
Verify the complete architecture.
IMPORTANT SECURITY AND CLEANUP NOTES
Keep SSH restricted to your current public IP /32 when SSH is needed.
Keep Block Public Access enabled on the S3 bucket.
Never place AWS secret keys or database passwords in source code.
Use IAM roles for EC2 instead of long-lived AWS access keys.
Keep the RDS database private.
A private subnet without a NAT Gateway or other suitable egress path may not be able to download packages or reach public AWS endpoints.
Check the AWS pricing for EC2, RDS, NAT Gateway, and other resources before starting the lab.
After the demonstration, remove disposable resources that you no longer need.
Empty versioned S3 buckets, including their object versions and delete markers, before deleting the buckets.
Confirm that required data has been backed up before deleting resources.

Standard S3 bucket name used throughout Week 3:

week3-core-cloud-lab-2026-vinayak
