### Managing Cloud cost effetively is crucial, especially for non-production environments where resource are often left running unnecessarily. By implementing an automated start and stop schedule for non-production instance, Organizations can save up to 30% on usage costs.

### Although this is a basic implementation, it is incredible effective and proactive. The True power lies not just starting and stopping instances, but in how automation can significantly improve operational efficiency and reduce manual overhead.

### This solution combines serveral AWS Services:
- Amazon EvenBridge Scheduler to trigger actions based on defined schedules.
- AWS Lambda Functions to execute start/stop operations with conditional logic.
- Amazon SNS Topics to send real-time notifications via email, Telegrams, or other communication channels.

## Let deepdive into step by step configuration.

### Step 1: Create an IAM Role for Lambda
Your Lambda function will need permissions to start and stop EC2 instances.

Go to IAM in AWS Console.

Create a new role with:

AWSLambdaBasicExecutionRole

AmazonEC2FullAccess (for simplicity, but should be limited to necessary actions).

Attach this role to your Lambda function in later steps.

### Step 2: Create an AWS Lambda Function
Your Lambda function will handle the start/stop operations based on the EventBridge trigger.

#### 1. Navigate to AWS Lambda
Go to AWS Lambda in the console.

Click Create Function → Choose Author from scratch.

Name it "EC2StartStopScheduler" and select Python runtime.

Assign the IAM role created in Step 1.

#### 2. Use the Python Script Below
Modify the function to start or stop EC2 instances:
#
python
import boto3
import os

 
ec2 = boto3.client('ec2') --- Define AWS EC2 client

INSTANCE_IDS = ['i-xxxxxxxxxxxxxxxxx'] --- Instance IDs to control (update these!)

def lambda_handler(event, context):
    action = event.get('action', 'stop')  ---- Default action: stop
    
    if action == 'start':
        ec2.start_instances(InstanceIds=INSTANCE_IDS)
        message = f"Started instances: {INSTANCE_IDS}"
    else:
        ec2.stop_instances(InstanceIds=INSTANCE_IDS)
        message = f"Stopped instances: {INSTANCE_IDS}"
    
    if 'sns_topic_arn' in event:
        sns = boto3.client('sns')
        sns.publish(TopicArn=event['sns_topic_arn'], Message=message) ---- Notify via SNS if configured

    return {"status": "success", "message": message}
#
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

json
{
  "action": "start"
}
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

json
{
  "action": "stop",
  "sns_topic_arn": "arn:aws:sns:region:account-id:EC2StartStopNotifications"
}


### Step 5: Test the Automation
Manually invoke the Lambda function from the AWS console with:
#
json
{"action": "start"}
(or { "action": "stop" } to stop instances.)
#
Ensure EventBridge triggers the function correctly.

Verify SNS notifications are delivered.

### Final Thoughts
With this setup: 
✔️ Automated cost-saving operations for non-production instances. 
✔️ Real-time notifications to keep track of instance state changes. 
✔️ Event-driven architecture without manual intervention.

#### Cheers!!
