# AWS Cost Optimization: Automated EC2 Start/Stop

## 📌 Overview
Managing AWS cloud costs is crucial, especially for non-production environments where instances may be left running unnecessarily. This solution **automates EC2 instance scheduling** using **Amazon EventBridge Scheduler**, **AWS Lambda**, and **SNS notifications**, potentially reducing costs by **up to 30%**.

## 🚀 Solution Components
This automation leverages several AWS services:
- **Amazon EventBridge Scheduler** → Triggers EC2 start/stop operations based on predefined schedules.
- **AWS Lambda Function** → Executes instance start/stop logic dynamically.
- **Amazon SNS** → Sends notifications via email, Telegram, or other communication channels.

---

## ⚙️ **Step-by-Step Setup Guide**

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
