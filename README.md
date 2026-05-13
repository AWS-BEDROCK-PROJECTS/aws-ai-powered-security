# aws-ai-powered-security
Serverless app security across seven layers using defense-in-depth, AWS-native services, and AI-driven automation for real-time threat detection, response, and remediation.

## Workshop Setup

1. Deploy the core services CloudFormation stack:

   ```
   aws cloudformation create-stack --stack-name MyCoreServicesStack --template-body file://cloudformation_iaac/#1-core-services.yaml --capabilities CAPABILITY_IAM --capabilities CAPABILITY_NAMED_IAM
   ```

2. Retrieve the API endpoint from the CloudFormation stack outputs:

   ```
   API_ENDPOINT=$(aws cloudformation describe-stacks \
     --stack-name Workshop-starter-app \
     --region us-east-1 \
     --query "Stacks[0].Outputs[?OutputKey=='APIEndpoint'].OutputValue" \
     --output text) && echo "API Endpoint: $API_ENDPOINT"
   ```

3. Test the health check endpoint:

   ```
   curl -s "$API_ENDPOINT/health"
   ```

4. Create a new order and capture the order ID:

   ```
   ORDER_ID=$(curl -s -X POST "$API_ENDPOINT/orders" \
     -H "Content-Type: application/json" \
     -d '{"customerId": "test-001", "items": [{"productId": "prod-001", "quantity": 1}]}' \
     | python3 -c "import sys, json; d=json.load(sys.stdin); print(d.get('orderId', 'ERROR: ' + json.dumps(d)))") && echo "Created order: $ORDER_ID"
   ```

5. Retrieve the order details:

   ```
   curl -s "$API_ENDPOINT/orders/$ORDER_ID" | jq .
   ```

## Authentication and Authorization

1. Retrieve the API Gateway REST API ID from the core services stack:

   ```
   API_ID=$(aws cloudformation describe-stacks \
     --stack-name Workshop-starter-app \
     --query "Stacks[0].Outputs[?OutputKey=='APIId'].OutputValue" \
     --output text \
     --region us-east-1) && echo "Starter App API ID: $API_ID"
   ```

2. Deploy the identity stack that creates Cognito User Pool and Identity Pool resources:

   ```
   aws cloudformation create-stack --stack-name Workshop-starter-app-identity --template-body file://cloudformation_iaac/#2-identity.yaml --capabilities CAPABILITY_IAM --capabilities CAPABILITY_NAMED_IAM --parameters ParameterKey=StarterAppAPIId,ParameterValue=$API_ID
   ```

3. Confirm the identity stack deployment status:

   ```
   aws cloudformation describe-stacks \
     --stack-name Workshop-starter-app-identity \
     --region us-east-1 \
     --query "Stacks[0].StackStatus" \
     --output text
   ```

4. Capture the Cognito User Pool ID from the identity stack outputs:

   ```
   USER_POOL_ID=$(aws cloudformation describe-stacks \
     --stack-name Workshop-starter-app-identity \
     --region us-east-1 \
     --query "Stacks[0].Outputs[?OutputKey=='UserPoolId'].OutputValue" \
     --output text)
   ```

5. Capture the Cognito User Pool client ID from the identity stack outputs:

   ```
   USER_POOL_CLIENT_ID=$(aws cloudformation describe-stacks \
     --stack-name Workshop-starter-app-identity \
     --region us-east-1 \
     --query "Stacks[0].Outputs[?OutputKey=='UserPoolClientId'].OutputValue" \
     --output text)
   ```

6. Print the Cognito User Pool information:

   ```
   echo "User Pool ID: $USER_POOL_ID |  User Pool Client ID: $USER_POOL_CLIENT_ID"
   ```

7. Create a test user in the Cognito User Pool:

   ```
   aws cognito-idp admin-create-user \
     --user-pool-id $USER_POOL_ID \
     --username workshop-test@example.com \
     --temporary-password "Workshop@Temp1" \
     --message-action SUPPRESS \
     --region us-east-1 \
     --output text \
     && echo "✓ Test user created: workshop-test@example.com"
   ```

8. Set a permanent password for the test user:

   ```
   aws cognito-idp admin-set-user-password \
     --user-pool-id $USER_POOL_ID \
     --username workshop-test@example.com \
     --password "Workshop@Perm1!" \
     --permanent \
     --region us-east-1 \
     && echo "✓ Permanent password set for test user."
   ```

9. Authenticate the test user and capture the token result:

   ```
   TOKEN_RESULT=$(aws cognito-idp initiate-auth \
     --auth-flow USER_PASSWORD_AUTH \
     --auth-parameters USERNAME=workshop-test@example.com,PASSWORD="Workshop@Perm1!" \
     --client-id $USER_POOL_CLIENT_ID \
     --region us-east-1)
   ```

10. Extract the ID token and verify authentication:

   ```
   ID_TOKEN=$(echo $TOKEN_RESULT | python3 -c \
     "import sys, json; print(json.load(sys.stdin)['AuthenticationResult']['IdToken'])")

   if [ -n "$ID_TOKEN" ]; then
     echo "✓ Authentication successful."
     echo ""
     echo "ID Token (first 50 characters):"
     echo "${ID_TOKEN:0:50}..."
   else
     echo "✗ Authentication failed. Check that Steps 3 and 4 completed successfully and that USER_POOL_CLIENT_ID is set."
   fi
   ```

11. List CloudFormation exports created by the identity stack:

   ```
   aws cloudformation list-exports \
     --region us-east-1 \
     --query "Exports[?starts_with(Name, 'Workshop-starter-app-identity')].[Name,Value]" \
     --output table
   ```
