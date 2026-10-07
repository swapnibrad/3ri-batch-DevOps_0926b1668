# AWS EC2 Running Instance Monitoring with Lambda and SNS

## ☁️ AWS Services Used

| AWS Service        | Purpose                           |
| ------------------ | --------------------------------- |
| Amazon EC2         | Hosts virtual machines            |
| AWS Lambda         | Executes Python monitoring code   |
| Amazon SNS         | Sends notifications through email |
| IAM                | Provides required permissions     |
| Amazon EventBridge | Automatically triggers Lambda     |
| CloudWatch Logs    | Stores Lambda execution logs      |

---

# 1. Create SNS Topic

Create an SNS topic:

```bash
aws sns create-topic \
    --name ec2-running-report
```

Example output:

```text
{
    "TopicArn": "arn:aws:sns:ap-south-1:123456789012:ec2-running-report"
}
```

Save the Topic ARN because it will be required by Lambda.

---

# 2. Subscribe Email to SNS

Subscribe an email address to the SNS topic:

```bash
aws sns subscribe \
    --topic-arn arn:aws:sns:ap-south-1:123456789012:ec2-running-report \
    --protocol email \
    --notification-endpoint your-email@example.com
```

Example:

```text
Protocol: email
Endpoint: your-email@example.com
```

SNS will send a confirmation email.

Open the email and click:

```text
Confirm subscription
```

Verify the subscription:

```bash
aws sns list-subscriptions-by-topic \
    --topic-arn arn:aws:sns:ap-south-1:123456789012:ec2-running-report
```

---

# 3. Create IAM Role for Lambda

Lambda requires permission to:

1. Read EC2 instance information.
2. Publish messages to SNS.
3. Write logs to CloudWatch.

Example IAM policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "sns:Publish"
            ],
            "Resource": "arn:aws:sns:ap-south-1:123456789012:ec2-running-report"
        }
    ]
}
```

Also attach the AWS managed policy:

```text
AWSLambdaBasicExecutionRole
```

---

# 4. Create Lambda Function

Create a Lambda function using:

```text
Runtime: Python 3.x
```

Function name:

```text
ec2-running-instance-report
```

---

# 5. Lambda Python Code

Create the Lambda function using the following code:

```python
import boto3
import os
from datetime import datetime

ec2 = boto3.client("ec2")
sns = boto3.client("sns")

SNS_TOPIC_ARN = os.environ["SNS_TOPIC_ARN"]


def lambda_handler(event, context):

    # Get running EC2 instances
    response = ec2.describe_instances(
        Filters=[
            {
                "Name": "instance-state-name",
                "Values": ["running"]
            }
        ]
    )

    instances = []

    for reservation in response["Reservations"]:
        for instance in reservation["Instances"]:

            instance_id = instance["InstanceId"]
            instance_type = instance["InstanceType"]
            private_ip = instance.get("PrivateIpAddress", "N/A")
            public_ip = instance.get("PublicIpAddress", "N/A")

            # Get EC2 Name tag
            name = "N/A"

            for tag in instance.get("Tags", []):
                if tag["Key"] == "Name":
                    name = tag["Value"]

            instances.append({
                "id": instance_id,
                "name": name,
                "type": instance_type,
                "private_ip": private_ip,
                "public_ip": public_ip
            })

    # Current report time
    current_time = datetime.utcnow().strftime(
        "%Y-%m-%d %H:%M:%S UTC"
    )

    # Generate report
    if instances:

        message = f"""
AWS EC2 RUNNING INSTANCE REPORT
================================

Report Time: {current_time}

Total Running Instances: {len(instances)}

"""

        for i, instance in enumerate(instances, start=1):

            message += f"""
Instance #{i}
-------------------------
Name         : {instance['name']}
Instance ID  : {instance['id']}
Instance Type: {instance['type']}
Private IP   : {instance['private_ip']}
Public IP    : {instance['public_ip']}
"""

    else:

        message = f"""
AWS EC2 RUNNING INSTANCE REPORT
================================

Report Time: {current_time}

No EC2 instances are currently running.
"""

    # Publish report to SNS
    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject="AWS EC2 Running Instance Report",
        Message=message
    )

    return {
        "statusCode": 200,
        "running_instances": len(instances),
        "message": "EC2 report sent successfully through SNS"
    }
```

---

# 6. Configure Lambda Environment Variable

Go to:

```text
Lambda
   ↓
Configuration
   ↓
Environment variables
```

Create:

```text
Key   : SNS_TOPIC_ARN

Value : arn:aws:sns:ap-south-1:123456789012:ec2-running-report
```

Replace the ARN with your actual SNS topic ARN.

---
---
---
# S3 to EC2 Automated File Deployment using AWS Lambda and AWS Systems Manager

## 1. Project Overview

This project demonstrates an automated file deployment mechanism where files uploaded to a specific folder (prefix) in an Amazon S3 bucket are automatically copied to an Amazon EC2 instance.

The solution uses:

* **Amazon S3** – Source location for application files
* **Amazon S3 Event Notification** – Detects newly uploaded files
* **AWS Lambda** – Processes the S3 event
* **AWS Systems Manager (SSM)** – Executes commands on the EC2 instance
* **Amazon EC2** – Destination server

### Automation Flow

```text
Developer / User
       |
       | Upload File
       v
+-----------------------+
|     Amazon S3         |
|                       |
| application/          |
|   ├── app.jar         |
|   ├── config.json     |
|   └── test.txt        |
+-----------+-----------+
            |
            | Object Created Event
            v
+-----------------------+
|     AWS Lambda        |
|                       |
| Read S3 Event         |
| Identify File         |
| Call SSM SendCommand  |
+-----------+-----------+
            |
            | AWS Systems Manager
            v
+-----------------------+
|       EC2 Instance    |
|                       |
| /opt/application/     |
|   ├── app.jar         |
|   ├── config.json     |
|   └── test.txt        |
+-----------------------+
```

---

# 2. Use Case

Consider a Java application running on an EC2 server.

Whenever a new application package is uploaded to:

```text
s3://my-application-bucket/application/
```

the file should automatically be deployed to:

```text
/opt/application/
```

on the EC2 instance.

For example:

```text
Upload:
s3://my-application-bucket/application/app.jar

        ↓

S3 Object Created Event

        ↓

Lambda

        ↓

AWS Systems Manager

        ↓

EC2

        ↓

/opt/application/app.jar
```

No manual SSH connection is required.

---

# 3. AWS Services Used

| AWS Service         | Purpose                       |
| ------------------- | ----------------------------- |
| Amazon S3           | Stores application files      |
| AWS Lambda          | Processes S3 events           |
| AWS Systems Manager | Executes commands on EC2      |
| Amazon EC2          | Destination server            |
| IAM                 | Provides required permissions |
| CloudWatch Logs     | Stores Lambda execution logs  |

---

# 4. Prerequisites

Before starting, make sure you have:

* AWS Account
* An S3 bucket
* An EC2 instance
* AWS Systems Manager Agent installed/running on EC2
* EC2 instance registered with Systems Manager
* IAM permissions to create/configure Lambda, S3 and IAM resources
* AWS CLI configured if using CLI-based verification

---

# 5. Example Configuration

For this project, we will use:

```text
S3 Bucket:
my-application-bucket

S3 Prefix:
application/

EC2 Instance:
i-0123456789abcdef0

EC2 Destination:
 /opt/application/
```

You should replace these values with your own environment.

---

# 6. Step 1 – Create S3 Bucket

Create an S3 bucket from the AWS Management Console.

Example:

```text
my-application-bucket
```

Inside the bucket, create a folder/prefix:

```text
application/
```

The final structure can look like:

```text
my-application-bucket
│
└── application/
      ├── app.jar
      ├── config.json
      └── test.txt
```

---

# 7. Step 2 – Prepare the EC2 Instance

The EC2 instance must be managed by AWS Systems Manager.

Verify the instance:

```bash
aws ssm describe-instance-information
```

You should see your EC2 instance in the output.

Example:

```text
InstanceId: i-0123456789abcdef0
PingStatus: Online
```

---

# 8. Step 3 – Attach SSM IAM Role to EC2

The EC2 instance needs permission to communicate with Systems Manager.

Attach the AWS managed policy:

```text
AmazonSSMManagedInstanceCore
```

### IAM Role

Example:

```text
EC2-SSM-Role
```

Attach:

```text
AmazonSSMManagedInstanceCore
```

After attaching the role, verify that the instance is visible in:

```text
AWS Console
→ Systems Manager
→ Fleet Manager
```

The instance should show:

```text
Ping status: Online
```

---

# 9. Step 4 – Give EC2 Permission to Read S3

The EC2 instance needs permission to download files from the S3 bucket.

Create an IAM policy similar to:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::my-application-bucket/application/*"
        }
    ]
}
```

Attach this policy to the EC2 IAM role.

### Important

The permission is restricted to:

```text
application/*
```

instead of allowing access to the entire S3 bucket.

---

# 10. Step 5 – Verify AWS CLI on EC2

Connect to the EC2 instance or use Systems Manager Session Manager.

Check AWS CLI:

```bash
aws --version
```

Example:

```text
aws-cli/2.x.x
```

Test S3 access:

```bash
aws s3 ls s3://my-application-bucket/application/
```

If the command successfully lists the files, the EC2 instance has the required S3 permission.

---

# 11. Step 6 – Create Lambda Function

Go to:

```text
AWS Console
→ Lambda
→ Functions
→ Create function
```

Select:

```text
Author from scratch
```

Example:

```text
Function name:
s3-to-ec2-ssm
```

Runtime:

```text
Python 3.x
```

Architecture:

```text
x86_64
```

Create the Lambda function.

---

# 12. Step 7 – Configure Lambda Environment Variables

Go to:

```text
Lambda
→ Configuration
→ Environment variables
```

Add:

| Variable          | Value                 |
| ----------------- | --------------------- |
| `EC2_INSTANCE_ID` | `i-0123456789abcdef0` |
| `DESTINATION`     | `/opt/application`    |

Example:

```text
EC2_INSTANCE_ID=i-0123456789abcdef0
DESTINATION=/opt/application
```

The S3 bucket does not need to be hardcoded because the bucket name is received from the S3 event.

---

# 13. Step 8 – Lambda Execution Role

The Lambda function needs permission to send commands to Systems Manager.

The minimum required permission includes:

```text
ssm:SendCommand
```

Example IAM policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:SendCommand"
            ],
            "Resource": "*"
        }
    ]
}
```

Lambda should also have CloudWatch logging permissions.

The easiest option is to attach:

```text
AWSLambdaBasicExecutionRole
```

---

# 14. Recommended Restricted Lambda IAM Policy

For a production environment, restrict the Systems Manager command to the required EC2 instance and document.

Example:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:SendCommand"
            ],
            "Resource": [
                "arn:aws:ec2:ap-south-1:123456789012:instance/i-0123456789abcdef0"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssm:SendCommand"
            ],
            "Resource": [
                "arn:aws:ssm:ap-south-1:*:document/AWS-RunShellScript"
            ]
        }
    ]
}
```

Replace:

```text
ap-south-1
123456789012
i-0123456789abcdef0
```

with your AWS Region, Account ID and EC2 Instance ID.

---

# 15. Step 9 – Add S3 Trigger to Lambda

This is the most important configuration.

Open:

```text
AWS Console
→ Lambda
→ Functions
→ s3-to-ec2-ssm
```

Under:

```text
Function overview
```

click:

```text
Add trigger
```

Select:

```text
S3
```

Configure:

```text
Bucket:
my-application-bucket

Event type:
All object create events
```

Set the prefix:

```text
application/
```

Optional suffix:

```text
.jar
```

For example, if you configure:

```text
Prefix:
application/

Suffix:
.jar
```

then:

```text
application/app.jar
```

will trigger Lambda.

But:

```text
application/config.json
application/test.txt
```

will not trigger Lambda.

If you want **all files** inside `application/` to trigger Lambda, configure only:

```text
Prefix:
application/
```

and leave suffix empty.

---

# 16. S3 Trigger Configuration

Recommended configuration:

```text
Trigger:
S3

Bucket:
my-application-bucket

Event type:
All object create events

Prefix:
application/

Suffix:
(optional)
```

The Lambda console normally adds the required S3 permission allowing S3 to invoke the Lambda function.

---

# 17. Step 10 – Lambda Python Function

Replace the Lambda function code with:

```python
import boto3
import os
import urllib.parse
import shlex

ssm = boto3.client("ssm")

EC2_INSTANCE_ID = os.environ["EC2_INSTANCE_ID"]
DESTINATION = os.environ["DESTINATION"]


def lambda_handler(event, context):

    print("S3 event received")

    for record in event["Records"]:

        bucket = record["s3"]["bucket"]["name"]

        key = urllib.parse.unquote_plus(
            record["s3"]["object"]["key"]
        )

        filename = os.path.basename(key)

        destination_file = f"{DESTINATION}/{filename}"

        s3_uri = f"s3://{bucket}/{key}"

        command = (
            f"sudo mkdir -p {shlex.quote(DESTINATION)} && "
            f"aws s3 cp "
            f"{shlex.quote(s3_uri)} "
            f"{shlex.quote(destination_file)}"
        )

        print(f"Bucket: {bucket}")
        print(f"S3 Key: {key}")
        print(f"Destination: {destination_file}")

        response = ssm.send_command(
            InstanceIds=[EC2_INSTANCE_ID],
            DocumentName="AWS-RunShellScript",
            Parameters={
                "commands": [command]
            },
            Comment=f"Copy {filename} from S3 to EC2"
        )

        command_id = response["Command"]["CommandId"]

        print(f"SSM Command ID: {command_id}")

    return {
        "statusCode": 200,
        "message": "File deployment initiated successfully"
    }
```

---

# 18. How the Lambda Function Works

When a file is uploaded:

```text
s3://my-application-bucket/application/app.jar
```

S3 generates an Object Created event.

The Lambda function receives an event similar to:

```json
{
    "Records": [
        {
            "s3": {
                "bucket": {
                    "name": "my-application-bucket"
                },
                "object": {
                    "key": "application/app.jar"
                }
            }
        }
    ]
}
```

Lambda extracts:

```text
Bucket:
my-application-bucket

Key:
application/app.jar
```

It then creates an SSM command:

```bash
sudo mkdir -p /opt/application

aws s3 cp \
s3://my-application-bucket/application/app.jar \
/opt/application/app.jar
```

The command is sent to EC2 through:

```python
ssm.send_command()
```

---

# 19. AWS Systems Manager Document

The Lambda function uses the AWS-managed Systems Manager document:

```text
AWS-RunShellScript
```

This document allows Lambda to execute shell commands on the target EC2 instance.

The important part of the Lambda code is:

```python
response = ssm.send_command(
    InstanceIds=[EC2_INSTANCE_ID],
    DocumentName="AWS-RunShellScript",
    Parameters={
        "commands": [command]
    }
)
```

---

# 20. Step 11 – Test the Automation

Create a test file:

```bash
echo "Hello from S3 automation" > test.txt
```

Upload it:

```bash
aws s3 cp test.txt \
s3://my-application-bucket/application/test.txt
```

Expected flow:

```text
test.txt
   |
   v
S3
   |
   | Object Created
   v
Lambda
   |
   | SendCommand
   v
SSM
   |
   v
EC2
   |
   v
/opt/application/test.txt
```

---

# 21. Verify File on EC2

Connect to EC2 using Session Manager.

Run:

```bash
ls -l /opt/application/
```

Expected:

```text
test.txt
```

Check the contents:

```bash
cat /opt/application/test.txt
```

Expected:

```text
Hello from S3 automation
```

---

# 22. Verify Lambda Logs

Go to:

```text
AWS Console
→ Lambda
→ s3-to-ec2-ssm
→ Monitor
→ View CloudWatch logs
```

You should see messages similar to:

```text
S3 event received

Bucket: my-application-bucket

S3 Key: application/test.txt

Destination: /opt/application/test.txt

SSM Command ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

---

# 23. Verify SSM Command

Go to:

```text
AWS Console
→ Systems Manager
→ Run Command
```

Find the command generated by Lambda.

You should see:

```text
Document:
AWS-RunShellScript

Target:
i-0123456789abcdef0

Status:
Success
```

You can also use the AWS CLI:

```bash
aws ssm list-commands
```

---

# 24. Complete Execution Flow

The complete execution sequence is:

### Step 1

User uploads:

```text
app.jar
```

to:

```text
s3://my-application-bucket/application/
```

### Step 2

S3 generates:

```text
Object Created Event
```

### Step 3

S3 invokes:

```text
s3-to-ec2-ssm Lambda
```

### Step 4

Lambda extracts:

```text
Bucket Name
Object Key
File Name
```

### Step 5

Lambda calls:

```text
AWS Systems Manager SendCommand
```

### Step 6

SSM sends the shell command to EC2.

### Step 7

EC2 executes:

```bash
aws s3 cp s3://my-application-bucket/application/app.jar /opt/application/app.jar
```

### Step 8

The file becomes available on EC2:

```text
/opt/application/app.jar
```

---

# 25. Complete Architecture

```text
                         AWS CLOUD
+---------------------------------------------------------------+
|                                                               |
|  +------------------+                                         |
|  |      Amazon S3   |                                         |
|  |                  |                                         |
|  | application/     |                                         |
|  |   app.jar        |                                         |
|  |   config.json     |                                         |
|  +--------+---------+                                         |
|           |                                                   |
|           | Object Created                                    |
|           v                                                   |
|  +------------------+                                         |
|  |   AWS Lambda     |                                         |
|  |                  |                                         |
|  | Python + boto3   |                                         |
|  +--------+---------+                                         |
|           |                                                   |
|           | ssm.send_command()                               |
|           v                                                   |
|  +--------------------------+                                 |
|  | AWS Systems Manager      |                                 |
|  |                          |                                 |
|  | AWS-RunShellScript       |                                 |
|  +------------+-------------+                                 |
|               |                                               |
|               | Execute command                               |
|               v                                               |
|       +--------------------+                                  |
|       |      EC2           |                                  |
|       |                    |                                  |
|       | /opt/application/  |                                  |
|       |                    |                                  |
|       | app.jar            |                                  |
|       | config.json        |                                  |
|       +--------------------+                                  |
|                                                               |
+---------------------------------------------------------------+
```

---

# 26. Important IAM Permissions

## Lambda

Lambda requires:

```text
ssm:SendCommand
```

and CloudWatch Logs permissions.

Minimum logging policy:

```text
AWSLambdaBasicExecutionRole
```

---

## EC2

EC2 requires:

```text
AmazonSSMManagedInstanceCore
```

and S3 read permission:

```text
s3:GetObject
```

for:

```text
arn:aws:s3:::my-application-bucket/application/*
```

---

# 27. Security Best Practices

Avoid giving unrestricted permissions such as:

```json
"Resource": "*"
```

in production whenever possible.

Instead, restrict access to:

```text
Specific S3 bucket
Specific S3 prefix
Specific EC2 instance
Specific SSM document
```

Recommended access flow:

```text
Lambda
   |
   | ssm:SendCommand
   v
Specific EC2

EC2
   |
   | s3:GetObject
   v
Specific S3 Prefix
```

---

# 28. Important S3 Trigger Consideration

Do not configure the Lambda trigger on a prefix where Lambda itself uploads files to S3.

For example, avoid:

```text
S3 application/
       |
       v
Lambda
       |
       v
Upload back to application/
```

This can create recursive Lambda invocations.

For this project, Lambda only reads the S3 object and sends the deployment command to EC2, so there is no recursive S3 trigger.

---

# 29. File Naming Example

The following files will trigger Lambda if only the prefix is configured:

```text
application/app.jar
application/config.json
application/index.html
application/script.sh
application/test.txt
```

Files outside the prefix will not trigger the Lambda:

```text
backup/app.jar
logs/test.log
documents/file.pdf
```

---

# 30. Troubleshooting

## Problem 1 – Lambda is not triggered

Check:

```text
Lambda
→ Configuration
→ Triggers
```

Verify that the S3 trigger exists.

Also verify the prefix:

```text
application/
```

---

## Problem 2 – Lambda executes but SSM command fails

Check Lambda IAM permissions:

```text
ssm:SendCommand
```

Also verify the EC2 instance is online in Systems Manager.

Run:

```bash
aws ssm describe-instance-information
```

---

## Problem 3 – SSM shows "TargetNotConnected"

Check:

* EC2 IAM role
* SSM Agent
* Network connectivity
* Systems Manager registration

The EC2 instance should show:

```text
PingStatus: Online
```

---

## Problem 4 – S3 AccessDenied on EC2

Verify the EC2 IAM role has:

```text
s3:GetObject
```

for:

```text
arn:aws:s3:::my-application-bucket/application/*
```

Test:

```bash
aws s3 cp \
s3://my-application-bucket/application/test.txt \
/tmp/test.txt
```

---

## Problem 5 – Destination directory does not exist

The Lambda command automatically creates it:

```bash
sudo mkdir -p /opt/application
```

You can manually verify:

```bash
ls -ld /opt/application
```

---

# 31. Useful AWS CLI Commands

### List S3 objects

```bash
aws s3 ls s3://my-application-bucket/application/
```

### Upload file

```bash
aws s3 cp app.jar \
s3://my-application-bucket/application/app.jar
```

### Check Lambda

```bash
aws lambda get-function \
--function-name s3-to-ec2-ssm
```

### Check SSM managed instances

```bash
aws ssm describe-instance-information
```

### List SSM commands

```bash
aws ssm list-commands
```

### Get command details

```bash
aws ssm get-command-invocation \
--command-id COMMAND_ID \
--instance-id i-0123456789abcdef0
```

---

# 32. Example Real-World Deployment

This architecture can be used for simple application deployment.

For example:

```text
Developer
    |
    | Build application
    v
Jenkins / CI Pipeline
    |
    | Upload app.jar
    v
S3
    |
    | S3 Event
    v
Lambda
    |
    | SSM
    v
EC2
    |
    v
/opt/application/app.jar
```

This allows an application artifact to be automatically copied to an EC2 server without requiring an SSH-based deployment.

---

# 33. Possible Enhancements

The project can be extended with:

### 1. Multiple EC2 Instances

Instead of one:

```text
EC2_INSTANCE_ID
```

Lambda can target multiple instances.

### 2. Auto Deployment

After copying the file, SSM can execute:

```bash
sudo systemctl restart myapp
```

### 3. Backup Existing Application

Before deployment:

```bash
cp app.jar app.jar.backup
```

### 4. Deployment Logging

Store deployment status in:

```text
CloudWatch
```

### 5. Failure Notification

Use:

```text
SNS
```

to send an email when deployment fails.

### 6. CI/CD Integration

Integrate:

```text
GitHub
    ↓
Jenkins
    ↓
Build
    ↓
S3
    ↓
Lambda
    ↓
SSM
    ↓
EC2
```

---

# 34. Project Structure

```text
s3-to-ec2-automation/
│
├── lambda_function.py
│
├── iam/
│   ├── lambda-policy.json
│   └── ec2-s3-policy.json
│
└── README.md
```

---

# 35. Final Architecture Summary

```text
                 FILE DEPLOYMENT AUTOMATION

Developer
    |
    | Upload
    v
+----------------------+
| Amazon S3            |
| application/         |
+----------+-----------+
           |
           | Object Created
           v
+----------------------+
| AWS Lambda           |
| Python / boto3       |
+----------+-----------+
           |
           | SendCommand
           v
+----------------------+
| AWS Systems Manager  |
| AWS-RunShellScript   |
+----------+-----------+
           |
           | Execute
           v
+----------------------+
| Amazon EC2           |
|                      |
| /opt/application/    |
+----------------------+
```

## Result

Once the configuration is completed, the deployment becomes automatic:

```text
Upload file to S3
       ↓
S3 Event
       ↓
Lambda Triggered
       ↓
Lambda Calls SSM
       ↓
SSM Executes Command on EC2
       ↓
EC2 Downloads File
       ↓
File Available in /opt/application/
```

**No manual Lambda execution and no manual SSH-based file transfer are required.**
