# S3 Event Notification Setup

This project involves setting up a system that monitors an **S3 bucket** for new file uploads, triggers a **Lambda function** to process the uploaded files, sends a notification using **SNS**, and stores metadata in an **SQS** queue for further processing. Components include:

a. S3 Event Trigger  
b. Lambda  
c. SQS  
d. SNS  
e. Integration of S3 with other services  

![image](https://github.com/user-attachments/assets/d058cfb6-78bc-4b98-837e-ac2e0120bd0b)

## Steps

### Step 1: Create S3 Bucket

- AWS S3 Service
- Create an S3 bucket: `my-s3-notification-bucket`.

### Step 2: Create SQS Queue

- SQS Service
- Create a standard SQS queue named `s3-sqs-queue`.

### Step 3: Create an SNS Topic

- SNS Service
- Add Subscriptions to SNS Topic:
    - **Add Email protocol** - specify email address & confirm the email from your inbox.
    - **Add SQS protocol** & enter the ARN of the SQS queue created in Step 2.

### Step 4: Create a Lambda Function

1. **Create a Lambda function** named `s3-notification-lambda` with runtime Python 3.9.
    - Lambda Service
    - Create function >> Choose "Author from scratch".
    - Function name: `s3-notification-lambda`.
    - Runtime: Python 3.9 >> Click "Create function".
    
2. **Configure the Lambda function**:
    - From the "Configuration" tab.
    - Click "Permissions" and attach the following policies to the Lambda execution role:
        - `AmazonSNSFullAccess`
        - `AmazonSQSFullAccess`
        
3. **Upload the Lambda code**:
    - Click the "Code" tab.
    - Click "Upload from" and select "zip file".
    - Upload a zipped file containing the following code (ensure the zipped file is named appropriately):

```python
import json
import boto3

s3_client = boto3.client('s3')
sns_client = boto3.client('sns')
sqs_client = boto3.client('sqs')

def lambda_handler(event, context):
    sns_topic_arn = 'arn:aws:sns:us-east-1:920668684836:s3-sns-topic'

    # Define the SQS queue URL
    sqs_queue_url = 'https://sqs.us-east-1.amazonaws.com/920668684836/s3-sqs-queue'

    # Process S3 event records
    for record in event['Records']:
        print(event)
        # Extract S3 bucket and object information
        s3_bucket = record['s3']['bucket']['name']
        s3_key = record['s3']['object']['key']
        
        # Example: Sending metadata to SQS
        metadata = {
            'bucket': s3_bucket,
            'key': s3_key,
            'timestamp': record['eventTime'],
            'action': record['eventName']
        }
        
        # Send metadata to SQS
        sqs_response = sqs_client.send_message(
            QueueUrl=sqs_queue_url,
            MessageBody=json.dumps(metadata)
        )
        
        # Example: Sending a notification to SNS
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
### Step 5: Create S3 Event Notification
1. Create an event notification for the S3 bucket.
   - From the S3 service, open my-s3-notification-bucket.
   - Click on the "Properties" tab.
   - Scroll down to "Event notifications" and click "Create event notification".
   - Name the event notification.
   - Under "Event types", select both "All object create events" and "All object removal events".
   - Under "Destination", select "Lambda function" and choose s3-notification-lambda.
   - Click "Save".

### Step 6: Test the Setup
1. Upload an object to the S3 bucket:
   - Go to the S3 service and open my-s3-notification-bucket.
   - Click "Upload" and add a file.
   - Click "Upload".

2. Check Lambda Execution:
   - Go to the Lambda service and open s3-notification-lambda.
   - Go to the "Monitor" tab and check the CloudWatch Logs for successful execution.

3. Verify SNS and SQS:
   - Check the email for the SNS notification.
   - Go to the SQS service and open s3-sqs-queue to verify the received messages.
