serverless rest api 

how it works

1. the user requests static frontend assets hosted on amazon s3 through amazon cloudfront.
2. the user authenticates via an amazon cognito user pool, which validates credentials and returns a secure json web token access or id token.
3. the user sends https api requests containing the bearer token to amazon api gateway.
4. aws waf inspects incoming traffic at the edge to block malicious payloads, mitigate automated threats, and enforce ip rate limits.
5. amazon api gateway validates the jwt token signature natively using the built-in cognito authorizer.
6. api gateway invokes the backend aws lambda function using proxy integration.
7. the lambda function executes crud operations against the amazon dynamodb table, strictly scoped to the caller's unique user id.
8. telemetry data, access logs, and distributed trace segments are streamed continuously to amazon cloudwatch and aws x-ray.

tech stack and aws services

- amazon s3 and cloudfront: static file hosting and global content delivery via edge locations.
- aws waf: edge security enforcing owasp top 10 rules and strict ip rate-limiting.
- amazon cognito: identity directory, user sign-up and sign-in flows, and secure jwt token issuance.
- amazon api gateway: fully managed rest api endpoints with native cognito authorizer integration and request validation.
- aws lambda (python 3.12): serverless backend compute running core business logic.
- amazon dynamodb: single-table nosql data store configured with on-demand capacity.
- aws x-ray and cloudwatch: application performance monitoring, log aggregation, and end-to-end distributed tracing.

database design

the project uses a single dynamodb table designed for optimal access patterns and strict tenant isolation:
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
