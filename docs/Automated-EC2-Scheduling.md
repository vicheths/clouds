# AWS Cost Optimization: Automated EC2 Start/Stop

## Overview
Managing AWS cloud costs is crucial, especially for non-production environments where instances may be left running unnecessarily. This solution **automates EC2 instance scheduling** using **Amazon EventBridge Scheduler**, **AWS Lambda**, and **SNS notifications**, potentially reducing costs by **up to 30%**.

## Solution Components
This automation leverages several AWS services:
- **Amazon EventBridge Scheduler** → Triggers EC2 start/stop operations based on predefined schedules.
- **AWS Lambda Function** → Executes instance start/stop logic dynamically.
- **Amazon SNS** → Sends notifications via email, Telegram, or other communication channels.

---

## **Step-by-Step Setup Guide**

### **Step 1: IAM Role Configuration**
1. Create an **IAM Role** with the following permissions:
   - `AWSLambdaBasicExecutionRole`
   - `AmazonEC2FullAccess` *(Adjust permissions for security best practices.)*
2. Attach this role to the **Lambda function** in the next step.

### **Step 2: Create AWS Lambda Function**
1. Navigate to **AWS Lambda** → Click **Create Function**.
2. Select **Python** as the runtime.
3. Use the following Python script:

```python
import boto3

ec2 = boto3.client('ec2')
INSTANCE_IDS = ['i-xxxxxxxxxxxxxxxxx']  # Replace with actual EC2 IDs

def lambda_handler(event, context):
    action = event.get('action', 'stop')  # Default action: stop
    if action == 'start':
        ec2.start_instances(InstanceIds=INSTANCE_IDS)
    else:
        ec2.stop_instances(InstanceIds=INSTANCE_IDS)

    return {"status": "success", "action": action, "instances": INSTANCE_IDS}
```
📌 Note: Replace "i-xxxxxxxxxxxxxxxxx" with actual EC2 instance IDs.

### Step 3: Create Amazon EventBridge Scheduler
This service will trigger your Lambda function at specified times.

Open Amazon EventBridge → Click Scheduler.

Create a new scheduler rule:

Schedule expression:
#
Start instances: cron(0 8 * * ? *) → (Daily at 8 AM UTC)

Stop instances: cron(0 20 * * ? *) → (Daily at 8 PM UTC)

#
Target → Select Lambda function (EC2StartStopScheduler).

In Input JSON, specify:
```python
json
{
  "action": "start"
}
```
(Create another scheduler with "action": "stop" for stopping instances.)

### Step 4: Set Up Amazon SNS for Notifications
If you want email or Telegram alerts, SNS can be useful.

#### 1. Create an SNS Topic
Go to Amazon SNS → Click Create Topic.
#
Name it "EC2StartStopNotifications".
#
#### 2. Subscribe to the Topic
Choose Email or Lambda (for further processing).

Confirm the subscription via your email.

#### 3. Modify Lambda Function to Publish Alerts
Update the Lambda function’s event JSON:
```python
json
{
  "action": "stop",
  "sns_topic_arn": "arn:aws:sns:region:account-id:EC2StartStopNotifications"
}
```

### Step 5: Test the Automation
Manually invoke the Lambda function from the AWS console with:
```python
json
{"action": "start"}
(or { "action": "stop" } to stop instances.)
```
Ensure EventBridge triggers the function correctly.

Verify SNS notifications are delivered.

### Final Thoughts
With this setup: 
- ✔️ Automated cost-saving operations for non-production instances. 
- ✔️ Real-time notifications to keep track of instance state changes. 
- ✔️ Event-driven architecture without manual intervention.

#### Cheers!!
