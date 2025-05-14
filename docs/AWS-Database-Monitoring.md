# AWS Database Monitoring: CloudWatch Alarms & Telegram Alerts

## Overview
This repository provides an **automated monitoring solution for AWS databases** using:
#
✔ **CloudWatch Alarms** → Monitors RDS metrics (CPU, storage, connections).  
✔ **Amazon SNS** → Triggers alerts when thresholds are exceeded.  
✔ **AWS Lambda** → Sends notifications to a **Telegram channel** for real-time response.  

## How It Works
1. **CloudWatch Alarms** detect abnormal database activity.
2. **SNS Topics** send alerts when thresholds are breached.
3. **Lambda Function** forwards alerts to Telegram via **bot API**.

---

## **Step-by-Step Setup Guide**

### 🔹 **Step 1: Configure Amazon SNS Topic**
1. Open **AWS SNS** → Click **Create Topic**.
2. Choose **Standard Topic** and name it `"DatabaseAlarmNotifications"`.
3. Copy the **SNS Topic ARN** for later.

### 🔹 **Step 2: Set Up CloudWatch Alarm for Database Metrics**
1. Open **AWS CloudWatch** → Click **Alarms** → **Create Alarm**.
2. Select **RDS Metric** (e.g., CPU utilization, free storage, connections).
3. Define thresholds:
   - **CPU Utilization** > 80%
   - **Free Storage Space** < 20GB
4. **Send Notification to SNS Topic** (`DatabaseAlarmNotifications`).

### 🔹 **Step 3: Create AWS Lambda Function to Send Telegram Alerts**
#### 1️⃣ **Create Lambda in AWS Console**
- Go to **AWS Lambda** → Click **Create Function**.
- Choose **Python 3.x** runtime.
- Assign an IAM role with:
  - `AWSLambdaBasicExecutionRole`
  - `AmazonSNSFullAccess`

#### 2️⃣ **Lambda Function Code (`telegram-alerts-lambda.py`)**
```python
import json
import boto3
import requests

SNS_CLIENT = boto3.client('sns')

TELEGRAM_BOT_TOKEN = "YOUR_TELEGRAM_BOT_TOKEN"
TELEGRAM_CHAT_ID = "YOUR_TELEGRAM_CHAT_ID"
TELEGRAM_API_URL = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"

def lambda_handler(event, context):
    message = event['Records'][0]['Sns']['Message']
    
    telegram_payload = {
        "chat_id": TELEGRAM_CHAT_ID,
        "text": f"🚨 AWS Database Alert 🚨\n{message}"
    }
    
    response = requests.post(TELEGRAM_API_URL, data=telegram_payload)
    
    return {
        "statusCode": 200,
        "body": json.dumps({
            "message": "Notification sent to Telegram",
            "telegram_response": response.json()
        })
    }
```
📌 Replace TELEGRAM_BOT_TOKEN and TELEGRAM_CHAT_ID with actual values.

###🔹 Step 4: Subscribe Lambda Function to SNS Topic
Go to **SNS Topic** → "DatabaseAlarmNotifications".

Click Create **Subscription**.

Choose **AWS Lambda** → Select "SendDatabaseAlertsToTelegram" function.

Click Create **Subscription**.

🎯 Testing & Validation
✅ Trigger a Test Alarm
Modify CloudWatch Alarm threshold temporarily.

Confirm SNS triggers Lambda.

Ensure a Telegram alert is received.

✅ Invoke Lambda Function Manually
Run this test event:
```python
json
{
  "Records": [
    {
      "Sns": {
        "Message": "🔥 High CPU Utilization detected on RDS instance!"
      }
    }
  ]
}
```
###
✔ Telegram should receive a formatted alert message.

#### Cheers !!
