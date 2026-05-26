# Serverless REST API — Lambda + API Gateway

**Services:** AWS Lambda · Amazon API Gateway · IAM Execution Roles · Amazon DynamoDB  
**Goal:** Build and deploy a fully serverless REST API that accepts HTTP requests, processes them with a Lambda function, and returns structured responses — no servers, no infrastructure management.

---

## What I Built

A serverless REST API with two endpoints: one to create a record and one to retrieve it. The full request lifecycle goes from the internet → API Gateway → Lambda → response, with IAM execution roles controlling what the Lambda function is allowed to do.

---

## Architecture

```
Client (HTTP Request)
        |
[API Gateway - REST API]
  POST /items  →  Lambda: createItem
  GET  /items/{id}  →  Lambda: getItem
        |
[Lambda Function]
  - Runtime: Python 3.12
  - Execution Role: LambdaAPIRole (least-privilege)
  - Permissions: dynamodb:PutItem, dynamodb:GetItem on specific table ARN
        |
[DynamoDB Table: Items]
  PK: itemId (String)
```

---

## Lambda Function — createItem

```python
import json
import boto3
import uuid

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Items')

def lambda_handler(event, context):
    body = json.loads(event['body'])
    item_id = str(uuid.uuid4())
    
    table.put_item(Item={
        'itemId': item_id,
        'name': body['name'],
        'description': body.get('description', '')
    })
    
    return {
        'statusCode': 201,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({'itemId': item_id, 'message': 'Item created'})
    }
```

---

## IAM Execution Role Policy

The Lambda function was granted only the minimum permissions it needs — access to two specific DynamoDB actions on one specific table. No wildcard permissions.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:ACCOUNT_ID:table/Items"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

---

## Key Configuration Decisions

**API Gateway integration type: Lambda Proxy**
Used Lambda Proxy integration so the full HTTP request (headers, body, path params) is passed directly to the function as an event object. The function is responsible for parsing and returning the correct HTTP response format.

**Scoped IAM execution role**
Lambda functions often get assigned `AmazonDynamoDBFullAccess` — that's over-permissioned. This function only needs `PutItem` and `GetItem` on one table. Scoping to a specific table ARN means a compromised function can't touch any other table in the account.

**CloudWatch Logs**
Lambda automatically creates a log group. The execution role includes `logs:PutLogEvents` so function output and errors are captured. In a real support scenario, CloudWatch logs are where you go first when a serverless function misbehaves.

---

## Troubleshooting Encountered

**Issue:** Lambda returned a 502 Bad Gateway from API Gateway.  
**Root cause:** The Lambda function was returning a plain string instead of the required response object format (statusCode, headers, body).  
**Fix:** Wrapped the return value in the proper API Gateway proxy response format with `statusCode`, `headers`, and `body` as a JSON string.  
**Lesson:** API Gateway Lambda Proxy integration requires a very specific response structure. Any deviation results in a 502 — it's not a Lambda error, it's a malformed response that API Gateway can't interpret.

**Issue:** Lambda function threw an `AccessDeniedException` when calling DynamoDB.  
**Root cause:** The execution role had the correct action (`PutItem`) but the `Resource` ARN had a typo in the table name.  
**Fix:** Corrected the ARN in the IAM policy. Verified with IAM Policy Simulator before re-deploying.  
**Lesson:** IAM errors are silent from the outside — the caller just sees an access denied. Always verify the exact ARN in your policy matches the actual resource ARN.

---

## What I'd Do Differently in Production

- Add **API Gateway request validation** to reject malformed requests before they reach Lambda
- Implement **API keys or Cognito authorizers** to authenticate callers
- Add **Lambda error handling and dead letter queues (DLQ)** for failed invocations
- Use **AWS SAM or CDK** to define and deploy this infrastructure as code instead of clicking through the console
- Enable **X-Ray tracing** across API Gateway and Lambda for end-to-end performance visibility
