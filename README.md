```markdown
serverless rest api with cognito, dynamodb and waf

graduation project for the aws solutions architect associate track.

this project implements a serverless rest api that lets authenticated users securely create, read, and delete personal records. it utilizes amazon api gateway, aws lambda, and amazon dynamodb, with user authentication handled by amazon cognito and perimeter edge protection managed by aws waf.

architecture diagram

![architecture diagram](MANARA PROJECT.png)[cite: 1]

how it works

1. the user requests static frontend assets hosted on amazon s3 through amazon cloudfront[cite: 1].
2. the user authenticates via an amazon cognito user pool, which validates credentials and returns a secure json web token access or id token[cite: 1].
3. the user sends https api requests containing the bearer token to amazon api gateway[cite: 1].
4. aws waf inspects incoming traffic at the edge to block malicious payloads, mitigate automated threats, and enforce ip rate limits[cite: 1].
5. amazon api gateway validates the jwt token signature natively using the built-in cognito authorizer[cite: 1].
6. api gateway invokes the backend aws lambda function using proxy integration[cite: 1].
7. the lambda function executes crud operations against the amazon dynamodb table, strictly scoped to the caller's unique user id[cite: 1].
8. telemetry data, access logs, and distributed trace segments are streamed continuously to amazon cloudwatch and aws x-ray[cite: 1].

tech stack and aws services

- amazon s3 and cloudfront: static website hosting and global content delivery via edge locations[cite: 1].
- aws waf: edge security enforcing owasp top 10 rules and strict ip rate-limiting[cite: 1].
- amazon cognito: identity directory, user sign-up and sign-in flows, and secure jwt token issuance[cite: 1].
- amazon api gateway: fully managed rest api endpoints with native cognito authorizer integration and request validation[cite: 1].
- aws lambda (python 3.12): serverless backend compute running core business logic[cite: 1].
- amazon dynamodb: single-table nosql data store configured with on-demand capacity[cite: 1].
- aws x-ray and cloudwatch: application performance monitoring, log aggregation, and end-to-end distributed tracing[cite: 1].

database design

the project uses a single dynamodb table designed for optimal access patterns and strict tenant isolation[cite: 1]:
- table name: serverless-records-table
- billing mode: on-demand (pay per request)
- partition key (userid): string (maps directly to the cognito sub claim)
- sort key (itemid): string (unique identifier for each individual record)
- item attributes: title (string), description (string), createdat (string)

api endpoints

- post /items: create a new item for the authenticated user (auth required: yes, bearer token)
- get /items: retrieve all items belonging to the caller (auth required: yes, bearer token)
- get /items/{id}: fetch a single item by id (auth required: yes, bearer token)
- delete /items/{id}: delete an item by id (auth required: yes, bearer token)

step-by-step deployment guide

1. create the dynamodb table
```bash
aws dynamodb create-table \
    --table-name serverless-records-table \
    --attribute-definitions \
        AttributeName=userId,AttributeType=S \
        AttributeName=itemId,AttributeType=S \
    --key-schema \
        AttributeName=userId,KeyType=HASH \
        AttributeName=itemId,KeyType=RANGE \
    --billing-mode PAY_PER_REQUEST

```

2. create cognito user pool and client

```bash
user_pool_id=$(aws cognito-idp create-user-pool \
    --pool-name serverless-api-userpool \
    --auto-verified-attributes email \
    --query 'UserPool.Id' --output text)

client_id=$(aws cognito-idp create-user-pool-client \
    --user-pool-id $user_pool_id \
    --client-name serverless-web-client \
    --no-generate-secret \
    --explicit-auth-flows USER_PASSWORD_AUTH \
    --query 'UserPoolClient.ClientId' --output text)

```

3. deploy the lambda backend

```bash
cd backend/
zip -r function.zip lambda_function.py

aws lambda create-function \
    --function-name records-api-handler \
    --runtime python3.12 \
    --handler lambda_function.lambda_handler \
    --role arn:aws:iam::<ACCOUNT_ID>:role/LambdaServerlessDynamoDBRole \
    --zip-file fileb://function.zip \
    --environment "Variables={TABLE_NAME=serverless-records-table}" \
    --tracing-config Mode=Active

```

4. configure api gateway rest api and authorizer

```bash
api_id=$(aws apigateway create-rest-api \
    --name records-service-api \
    --query 'id' --output text)

aws apigateway create-authorizer \
    --rest-api-id $api_id \
    --name CognitoAuth \
    --type COGNITO_USER_POOLS \
    --provider-arns arn:aws:cognito-idp:<REGION>:<ACCOUNT_ID>::userpool/$user_pool_id \
    --identity-source method.request.header.Authorization

```

testing the api

1. authenticate with cognito to retrieve a jwt token

```bash
token=$(aws cognito-idp initiate-auth \
    --auth-flow USER_PASSWORD_AUTH \
    --client-id $client_id \
    --auth-parameters USERNAME=testuser@example.com,PASSWORD=Password123! \
    --query 'AuthenticationResult.IdToken' --output text)

```

2. create a record

```bash
curl -X POST https://<api-id>.execute-api.<region>[.amazonaws.com/prod/items](https://.amazonaws.com/prod/items) \
     -H "Authorization: Bearer $token" \
     -H "Content-Type: application/json" \
     -d '{"itemId":"item_1","title":"test item","description":"testing serverless api"}'

```

3. retrieve all records

```bash
curl -X GET https://<api-id>.execute-api.<region>[.amazonaws.com/prod/items](https://.amazonaws.com/prod/items) \
     -H "Authorization: Bearer $token"

```

```

```
