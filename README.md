# S3 Event Notification Setup: SNS, SQS, Lambda

This repository provides a detailed guide on setting up S3 event notifications to trigger a Lambda function whenever an object is created or removed from an S3 bucket. The Lambda function processes the event and sends notifications to both an SNS topic and an SQS queue.

## Project Overview

This project demonstrates how to:
- Create S3 buckets
- Set up SQS and SNS for notifications
- Deploy a Lambda function triggered by S3 events
- Configure IAM roles and policies for the Lambda function

![image](https://github.com/user-attachments/assets/2964b947-16f8-4cae-9d99-d52c0b54b1ba)

## Prerequisites

- AWS CLI configured on your local machine
- Basic knowledge of AWS services (S3, Lambda, SQS, SNS, IAM)

## Steps

### Step 1: Create S3 Bucket

**Step 1. Create an S3 bucket:**

   ```bash
   aws s3api create-bucket --bucket my-s3-notification-bucket --region us-east-1
   ```
**Step 2: Create SQS Queue**
- Create a standard SQS queue:  
   ```bash
  aws sqs create-queue --queue-name s3-sqs-queue
   ```
**Step 3: Create SNS Topic**
1. Create an SNS topic:
  ```bash
  aws sns create-topic --name s3-sns-topic
  ```
2. Add subscriptions to the SNS topic:
   - Email subscription
   - SQS subscription
   ```bash
   # Subscribe an email
    aws sns subscribe --topic-arn arn:aws:sns:us-east-1:123456789012:s3-sns-topic --protocol email --notification-endpoint you@example.com
   # Subscribe an SQS queue
    aws sns subscribe --topic-arn arn:aws:sns:us-east-1:123456789012:s3-sns-topic --protocol sqs --notification-endpoint arn:aws:sqs:us-east-1:123456789012:s3-sqs-queue
   ```

**Step 4: Create Lambda Function**
1. Create a Lambda function:
   ```bash
   zip function.zip lambda_function.py
   aws lambda create-function --function-name s3-notification-lambda --zip-file fileb://function.zip --handler lambda_function.lambda_handler --runtime python3.9 --role arn:aws:iam::123456789012:role/lambda-role
    ```
  - lambda_function.py:
  ```bash
  import json
import boto3

s3_client = boto3.client('s3')
sns_client = boto3.client('sns')
sqs_client = boto3.client('sqs')

def lambda_handler(event, context):
    sns_topic_arn = 'arn:aws:sns:us-east-1:123456789012:s3-sns-topic'
    sqs_queue_url = 'https://sqs.us-east-1.amazonaws.com/123456789012/s3-sqs-queue'

    for record in event['Records']:
        print(event)
        s3_bucket = record['s3']['bucket']['name']
        s3_key = record['s3']['object']['key']
        
        metadata = {
            'bucket': s3_bucket,
            'key': s3_key,
            'timestamp': record['eventTime'],
            'action': record['eventName']
        }
        
        sqs_response = sqs_client.send_message(
            QueueUrl=sqs_queue_url,
            MessageBody=json.dumps(metadata)
        )
        
        notification_message = f"New file uploaded to S3 bucket '{s3_bucket}' with key '{s3_key}'"
        
        sns_response = sns_client.publish(
            TopicArn=sns_topic_arn,
            Message=notification_message,
            Subject="File Upload Notification"
        )

    return {
        'statusCode': 200,
        'body': json.dumps('Processing complete')
    }
```

2. Attach the following policies to the Lambda function's execution role:
   - ```AmazonSNSFullAccess```
   - ```AmazonSQSFullAccess```
  ```bash
  aws iam attach-role-policy --role-name lambda-role --policy-arn arn:aws:iam::aws:policy/AmazonSNSFullAccess
  aws iam attach-role-policy --role-name lambda-role --policy-arn arn:aws:iam::aws:policy/AmazonSQSFullAccess
  ```

**Step 5: Configure S3 Event Notification**
1. Set up event notifications for the S3 bucket:
    - Navigate to the S3 bucket in the AWS Management Console.
    - Under Properties, click Create Event Notification.
    - Configure the event types:
        * Object creation (All object create events)
        * Object removal (All object removal events)
        * Object restore (All restore object events)
        * Select the Lambda function s3-notification-lambda as the destination.
      
**Step 6: Test the Setup**
1. Upload an object to the S3 bucket:
   ```bash
   echo "Hello World" > test-file.txt
   aws s3 cp test-file.txt s3://my-s3-notification-bucket/
   ```

2. Check the Lambda function logs in CloudWatch:
    
   - Navigate to the Lambda function in the AWS Management Console.
   - Go to Monitor and select View CloudWatch Logs.

3. Verify that the notifications are sent to SNS and SQS:
   - Check the email for the SNS notification.
   - Check the SQS queue for the message.
