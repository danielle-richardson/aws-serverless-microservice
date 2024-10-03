# Serverless API with AWS Lambda, DynamoDB, and API Gateway

This project demonstrates how to build a serverless API using **AWS Lambda**, **DynamoDB**, and **API Gateway**. The API supports CRUD operations on a DynamoDB table, using Lambda as the backend logic.

---

## Overview and High-Level Design

The architecture consists of the following AWS services:

1. **Amazon API Gateway**: To expose the API.
2. **AWS Lambda**: To handle business logic for DynamoDB CRUD operations.
3. **Amazon DynamoDB**: To store data.
4. **CloudWatch Logs**: For logging Lambda execution.

In this tutorial, you will:
- Create an API with one resource (`DynamoDBManager`) and one method (`POST`).
- Invoke Lambda functions via API Gateway to perform operations on DynamoDB, including create, read, update, delete, and list items.

---

## Features

- **CRUD Operations** on DynamoDB via API Gateway:
  - Create, update, delete, read, and list items in a DynamoDB table.
- **Postman & Curl Support**: Use Postman or Curl to interact with the API.
- **Serverless**: Built entirely using AWS serverless components.

---

## Setup

### Step 1: Create a Lambda IAM Role

The Lambda function requires permissions to interact with **DynamoDB** for CRUD operations and **CloudWatch Logs** for logging. We will split these into two policies:

1. **Custom DynamoDB Policy**: Granting specific CRUD access to the DynamoDB table.
2. **AWS Managed CloudWatch Logs Policy**: Use the AWS-managed policy `AWSLambdaBasicExecutionRole` for logging permissions.

#### Step 1.1: Create a Custom DynamoDB Policy

Create a custom policy that grants **CRUD access** to your specific DynamoDB table (`lambda-apigateway`). Use the following policy JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DynamoDBAccess",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Scan"
      ],
      "Resource": "*"
    }
  ]
}
```
> **Disclaimer: Why `*` in Resource:**
>
> At this point, we haven’t created the DynamoDB table yet, so we don't have the specific Amazon Resource Name (ARN). For now, we'll allow access to all DynamoDB tables using `*`. Once the DynamoDB table is created, we will update this policy to target the exact ARN of the specific table.

#### Step 1.2: Attach AWS Managed CloudWatch Logs Policy

For logging permissions, we will use the AWS-managed policy **`AWSLambdaBasicExecutionRole`**, which provides the necessary permissions for **CloudWatch Logs** without needing to define a custom policy.

1. Go to the **IAM Console** and navigate to the **lambda-dynamodb-execution-role**.
2. Click **Add permissions**.
3. In the search bar, type **`AWSLambdaBasicExecutionRole`** and select the managed policy from the list.
4. Click **Attach policy**.

This AWS-managed policy allows Lambda to:
- **Create log groups**.
- **Create log streams**.
- **Put log events** (i.e., write logs to CloudWatch).

By attaching this managed policy, you ensure that your Lambda function has the appropriate permissions for CloudWatch logging without over-provisioning.

### Step 2: Create the Lambda Function

1. In the AWS Lambda Console, create a new function:
   - **Name**: `LambdaFunctionOverHttps`
   - **Runtime**: Python 3.12
   - **Permissions**: Attach the `lambda-dynamodb-execution-role` created earlier.

2. Replace the default code with the following:

    ```python
    import boto3
    import json

    def lambda_handler(event, context):
        operation = event['operation']
        
        if 'tableName' in event:
            dynamo = boto3.resource('dynamodb').Table(event['tableName'])
        
        operations = {
            'create': lambda x: dynamo.put_item(**x),
            'read': lambda x: dynamo.get_item(**x),
            'update': lambda x: dynamo.update_item(**x),
            'delete': lambda x: dynamo.delete_item(**x),
            'list': lambda x: dynamo.scan(**x),
            'echo': lambda x: x,
            'ping': lambda x: 'pong'
        }
        
        if operation in operations:
            return operations[operation](event.get('payload'))
        else:
            raise ValueError(f'Unrecognized operation "{operation}"')
    ```

### Step 3: Test the Lambda Function

Let's test the Lambda function by performing an echo operation before we connect it to DynamoDB.

1. In the Lambda Console, configure a test event using the following JSON:
    ```json
    {
        "operation": "echo",
        "payload": {
            "somekey1": "somevalue1",
            "somekey2": "somevalue2"
        }
    }
    ```

2. Run the test, and the output should echo back the payload.

---

## Create DynamoDB Table

Next, create the DynamoDB table that will be used by the Lambda function:

1. Go to the **DynamoDB Console**.
2. Choose **Create table**.
3. Set the following parameters:
   - **Table name**: `lambda-apigateway`
   - **Primary key**: `id` (string)
4. Click **Create**.

---

## Create the API in API Gateway

1. Go to the **API Gateway Console**.
2. Click **Create API**.
3. Select **REST API**, then click **Build**.
4. Name the API **DynamoDBOperations**, and click **Create API**.

### Create API Resources

1. Click **Actions** > **Create Resource**.
2. Set **Resource Name** to `DynamoDBManager`, and click **Create Resource**.

### Create POST Method for API

1. With the `/DynamoDBManager` resource selected, click **Actions** > **Create Method**.
2. Select **POST** from the dropdown, then select the **LambdaFunctionOverHttps** Lambda function.
3. Save the integration and allow API Gateway to invoke the Lambda function.

---

## Deploy the API

1. In **API Gateway**, click **Actions** > **Deploy API**.
2. Create a new stage called `Prod`, and click **Deploy**.

---

## Running the Solution

You can now use Postman or Curl to interact with your API and DynamoDB.

### Example Request - Create Item (Curl)

```bash
curl -X POST -d "{\"operation\":\"create\",\"tableName\":\"lambda-apigateway\",\"payload\":{\"Item\":{\"id\":\"1\",\"name\":\"Bob\"}}}" https://<API_ID>.execute-api.<REGION>.amazonaws.com/prod/DynamoDBManager
