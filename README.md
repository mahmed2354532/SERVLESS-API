```markdown
serverless cloud-native restful api with cognitive security and edge defense

aws solutions architect associate graduation project.

this repository showcases a production-ready serverless backend built to handle secure user-specific data operations. by leveraging fully managed aws services, the application achieves elastic scaling, millisecond data retrieval, and robust edge-to-database protection without provisioning any servers.

---

## architecture diagram

![architecture diagram](MANARA_PROJECT.png)

## workflow breakdown

1. static assets for the frontend interface are retrieved via amazon cloudfront from an amazon s3 bucket origin.
2. users authenticate through an amazon cognito user pool, obtaining a signed jwt bearer token upon successful verification.
3. incoming https requests hit the amazon api gateway endpoint carrying the authorization token.
4. aws waf inspects traffic at the network perimeter, mitigating potential layer-7 attacks and enforcing strict throttling rules.
5. api gateway authenticates the payload instantly using its built-in cognito authorizer integration.
6. validated requests trigger a purpose-built aws lambda compute function using proxy integration.
7. the python backend executes granular persistence actions against an amazon dynamodb table, isolating records per user.
8. operational insights, metrics, and trace maps are continuously forwarded to amazon cloudwatch and aws x-ray.

---

## core tech stack

- amazon s3 & cloudfront: high-performance static hosting and global content delivery network.
- aws waf: edge firewall providing owasp protection and request rate-limiting.
- amazon cognito: managed user identity directory and token issuer.
- amazon api gateway: fully managed rest routing layer with integrated authorizers.
- aws lambda (python 3.12): event-driven serverless computing runtime.
- amazon dynamodb: single-table nosql database with pay-per-request scaling.
- aws x-ray & cloudwatch: comprehensive distributed tracing and logging facilities.

---

## data model design

the database layer relies on a single dynamodb table optimized for single-digit millisecond latency and absolute data separation:

- table name: serverless-records-table
- billing mode: on-demand (pay per request)
- partition key (userid): string (derived directly from the cognito user sub claim)
- sort key (itemid): string (unique record identifier)
- attributes: title (string), description (string), createdat (string)

---

## rest api routes

- post /items: register a new record entry for the signed-in user (requires bearer token)
- get /items: fetch all records belonging to the current caller (requires bearer token)
- get /items/{id}: retrieve a specific record by its identifier (requires bearer token)
- delete /items/{id}: remove a specific record permanently (requires bearer token)

---

## deployment automation guide

### 1. provision the dynamodb table
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

### 2. set up amazon cognito

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

### 3. deploy the lambda compute function

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

### 4. configure api gateway and authorizer

```bash
api_id=$(aws apigateway create-rest-api \
    --name records-service-api \
    --query 'id' --output text)

aws apigateway create-authorizer \
    --rest-api-id $api_id \
    --name CognitoAuth \
    --type COGNITO_USER_POOLS \
    --provider-arns arn:aws:cognito-idp:<REGION>:<ACCOUNT_ID>:userpool/$user_pool_id \
    --identity-source method.request.header.Authorization

```

---

## verification and testing

### authenticate and fetch token

```bash
token=$(aws cognito-idp initiate-auth \
    --auth-flow USER_PASSWORD_AUTH \
    --client-id $client_id \
    --auth-parameters USERNAME=testuser@example.com,PASSWORD=Password123! \
    --query 'AuthenticationResult.IdToken' --output text)

```

### create a test record

```bash
curl -X POST https://<api-id>.execute-api.<region>[.amazonaws.com/prod/items](https://.amazonaws.com/prod/items) \
     -H "Authorization: Bearer $token" \
     -H "Content-Type: application/json" \
     -d '{"itemId":"item_1","title":"cloud architecture note","description":"testing serverless deployment"}'

```

### retrieve all records

```bash
curl -X GET https://<api-id>.execute-api.<region>[.amazonaws.com/prod/items](https://.amazonaws.com/prod/items) \
     -H "Authorization: Bearer $token"

```

```

```
