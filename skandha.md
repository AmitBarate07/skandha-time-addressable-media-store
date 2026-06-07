# Time-Addressable Media Store (TAMS) — Complete AWS Deployment Guide (From Scratch)

This guide assumes you are starting from zero — no AWS account, no tools installed, nothing. We will set up everything step by step on a Windows machine.

---No action needed account is alrady preapred 

## Part 1: Create an AWS Account

1. Go to [https://aws.amazon.com](https://aws.amazon.com) and click **Create an AWS Account**
2. Provide your email address, choose an account name, and set a password
3. Choose **Personal** or **Business** account type
4. Enter your contact information
5. Add a valid payment method (credit/debit card) — AWS has a free tier but requires a card on file
6. Complete identity verification (phone/SMS)
7. Select the **Basic Support (Free)** plan
8. Sign in to the [AWS Management Console](https://console.aws.amazon.com)

---

## Part 2: Create an IAM Admin User 

Do not use the root account for deployments. Create a dedicated IAM user.

1. Sign in to the AWS Console as root
2. Go to **IAM** → **Users** → **Create user**
3. Username: `tams-deployer`
4. Check **Provide user access to the AWS Management Console** (optional, for console access)
5. Click **Next**
6. Select **Attach policies directly**
7. Search and attach: `AdministratorAccess`
   > For production, scope this down. For a dev setup, Admin is fine.
8. Click **Next** → **Create user**
9. Now create access keys for CLI usage:
   - Click on the user `tams-deployer`
   - Go to **Security credentials** tab
   - Click **Create access key**
   - Select **Command Line Interface (CLI)**
   - Confirm and click **Create access key**
   - **Save the Access Key ID and Secret Access Key** — you will not see the secret again

---

## Part 3: Install Required Tools on Windows

### 3.1 Install Git

1. Download from [https://git-scm.com/download/win](https://git-scm.com/download/win)
2. Run the installer with default options
3. Verify:
   ```bash
   git --version
   ```

### 3.2 Install Python 3.14

1. Download from [https://www.python.org/downloads/](https://www.python.org/downloads/)
2. During installation, check **"Add Python to PATH"**
3. Verify:
   ```bash
   python --version
   pip --version
   ```

### 3.3 Install Poetry (Python Dependency Manager)

```bash
pip install poetry
```
Verify:
```bash
poetry --version
```

### 3.4 Install Docker Desktop

1. Download from [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/)
2. Install and restart your machine if prompted
3. Open Docker Desktop and ensure it is running
4. Verify:
   ```bash
   docker --version
   ```

> If you are on Windows Home, Docker Desktop requires WSL 2. Follow the prompts during installation to enable it.

### 3.5 Install AWS CLI

1. Download the MSI installer from [https://aws.amazon.com/cli/](https://aws.amazon.com/cli/)
2. Run the installer
3. Verify:
   ```bash
   aws --version
   ```

### 3.6 Configure AWS CLI

```bash
aws configure
```

Enter the following when prompted:

| Prompt | Value |
|--------|-------|
| AWS Access Key ID | Your `tams-deployer` access key |
| AWS Secret Access Key | Your `tams-deployer` secret key |
| Default region name | Your preferred region (e.g. `us-east-1`) |
| Default output format | `json` |

Verify it works:
```bash
aws sts get-caller-identity
```
You should see your account ID and user ARN.

### 3.7 Install AWS SAM CLI

1. Download the MSI installer from [https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)
2. Run the installer
3. Verify:
   ```bash
   sam --version
   ```

---

## Part 4: Configure API Gateway CloudWatch Logging (One-Time Per Region)

This is a mandatory prerequisite. The TAMS stack enables CloudWatch logging on its API Gateway stage. Without this role configured, the deployment will fail.

Reference: [AWS Documentation — Set up CloudWatch logging for REST APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-logging.html)

### 4.1 Verify Your AWS CLI is Configured

Make sure your CLI is working and pointing to the correct region:

```bash
aws sts get-caller-identity
```

You should see your account ID and user ARN. If not, run `aws configure` first (see Part 3.6).

### 4.2 Create the IAM Role for API Gateway CloudWatch Logging

First, create a trust policy file that allows API Gateway to assume this role:

```bash
echo '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"apigateway.amazonaws.com"},"Action":"sts:AssumeRole"}]}' > apigw-trust-policy.json
```

Now create the role:

```bash
aws iam create-role --role-name APIGatewayCloudWatchRole --assume-role-policy-document file://apigw-trust-policy.json
```

Attach the managed policy that grants CloudWatch Logs permissions:

```bash
aws iam attach-role-policy --role-name APIGatewayCloudWatchRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonAPIGatewayPushToCloudWatchLogs
```

Get the Role ARN (you'll need it in the next step):

```bash
aws iam get-role --role-name APIGatewayCloudWatchRole --query "Role.Arn" --output text
```

This will output something like: `arn:aws:iam::123456789012:role/APIGatewayCloudWatchRole`

### 4.3 Assign the Role to API Gateway

Use the Role ARN from the previous step to configure API Gateway's account-level CloudWatch logging:

```bash
aws apigateway update-account --patch-operations op=replace,path=/cloudwatchRoleArn,value=arn:aws:iam::674904383813:role/APIGatewayCloudWatchRole
```

> Replace `YOUR_ACCOUNT_ID` with your actual AWS account ID. You can get it with:
> ```bash
> aws sts get-caller-identity --query "Account" --output text
> ```

Or do it all in one shot (PowerShell):

```powershell
$ROLE_ARN = aws iam get-role --role-name APIGatewayCloudWatchRole --query "Role.Arn" --output text
aws apigateway update-account --patch-operations "op=replace,path=/cloudwatchRoleArn,value=$ROLE_ARN"
```

### 4.4 Verify the Configuration

```bash
aws apigateway get-account
```

You should see output like:

```json
{
    "cloudwatchRoleArn": "arn:aws:iam::123456789012:role/APIGatewayCloudWatchRole",
    "throttleSettings": {
        "burstLimit": 5000,
        "rateLimit": 10000.0
    },
    ...
}
```

### 4.5 Cleanup the Temp File

```bash
del apigw-trust-policy.json
```

> This must be done once per AWS Region where you plan to deploy. If you deploy TAMS in `us-east-1` and later in `eu-west-1`, you need to repeat steps 4.3 and 4.4 in `eu-west-1` (the IAM role is global, but the API Gateway account setting is per-region).

---

## Part 5: Clone and Prepare the Project

### 5.1 Clone the Repository

```bash
git clone https://github.com/awslabs/time-addressable-media-store.git
cd time-addressable-media-store
```

> **Recommended:** Use the latest stable release from the [Releases page](https://github.com/awslabs/time-addressable-media-store/releases):
> ```bash
> git checkout tags/<version>
> ```

### 5.2 Install Python Dependencies

```bash
poetry install
```

### 5.3 Generate the API Specification

This fetches the BBC TAMS API spec and generates the OpenAPI schema and Pydantic models:

```bash
make api-spec
```

> On Windows without `make`, you can install it via [chocolatey](https://chocolatey.org/):
> ```bash
> choco install make
> ```
> Or use WSL/Git Bash which includes `make`.

---

## Part 6: Build the Application

```bash
sam build --use-container
```

This pulls a Docker container matching the Lambda runtime (Python 3.14 on Amazon Linux) and builds all Lambda functions inside it. First run may take 5-10 minutes.

Verify the build succeeded — you should see:
```
Build Succeeded
```

---

## Part 7: Deploy to AWS

```bash
sam deploy --guided
```

### Deployment Parameters

You will be prompted for each parameter. Here is what to enter:

| Parameter | What to Enter | Notes |
|-----------|---------------|-------|
| **Stack Name** | `tams` | Unique name for your CloudFormation stack |
| **AWS Region** | `us-east-1` | Or your preferred region |
| **VpcId** | *(leave blank)* | A new VPC will be created for you |
| **VpcAZs** | *(leave blank)* | Auto-configured with new VPC |
| **PrivateSubnetIds** | *(leave blank)* | Auto-configured with new VPC |
| **NeptuneDBInstanceClass** | `db.serverless` | Serverless scales automatically |
| **NeptuneServerlessConfiguration** | `1,128` | Min and max Neptune capacity units |
| **DeployWaf** | `Yes` | Adds WAF protection to the API |
| **JwtIssuerUrl** | *(leave blank)* | Cognito will be deployed for auth |
| **LambdaAuthorizerArn** | *(leave blank)* | Default authorizer will be created |
| **Confirm changes before deploy** | `y` | Review before applying |
| **Allow SAM CLI IAM role creation** | `y` | Required for Lambda roles |
| **Disable rollback** | `n` | Keep rollback enabled |
| **Save arguments to samconfig.toml** | `y` | Saves config for future deploys |

### What Gets Created

The deployment creates:

- 1 VPC with 2 private subnets and VPC endpoints (S3, DynamoDB, SQS, Events, SSM)
- 1 Neptune serverless cluster (graph database)
- 3 DynamoDB tables (service, flow segments, flow storage)
- 1 S3 bucket (media storage)
- 5 SQS queues (delete requests, cleanup, webhook delivery, webhook DLQ, object duplication)
- 1 EventBridge custom event bus
- 12+ Lambda functions (API handlers, webhooks, cleanup workers)
- 1 API Gateway REST API with Lambda Authorizer
- 1 Cognito User Pool with App Client
- 1 WAF WebACL
- Associated IAM roles, security groups, and CloudWatch alarms

> **Expect 25-35 minutes** for the full deployment. Neptune alone takes ~20 minutes.

### Deployment Complete

When finished, SAM CLI prints the stack outputs:

```
Key                 Value
---                 -----
ApiEndpoint         https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/Prod
UserPoolId          us-east-1_XXXXXXXXX
TokenUrl            https://xxxxxxxxxx.auth.us-east-1.amazoncognito.com/oauth2/token
UserPoolClientId    xxxxxxxxxxxxxxxxxxxxxxxxxx
MediaStorageBucket  tams-mediastoragebucket-xxxxxxxxxxxx
NeptuneEndpoint     tams-neptunestack-xxx.cluster-xxxx.us-east-1.neptune.amazonaws.com
```

**Save these values.** You will need them to interact with the API.

---

## Part 8: Get Your API Credentials

### 8.1 Get the Client Secret

1. Go to **Cognito Console** → **User Pools**
2. Click on the pool matching your stack name
3. Go to **App integration** tab
4. Find the App Client named `{STACK_NAME}-CognitoStack-{XXX}-all-scopes`
5. Click on it and copy the **Client secret**

### 8.2 Request an Access Token

```bash
curl -X POST ^
  -d "client_id=YOUR_CLIENT_ID&client_secret=YOUR_CLIENT_SECRET&grant_type=client_credentials&scope=tams-api/read tams-api/write" ^
  https://YOUR_TOKEN_URL
```

The response will contain:
```json
{
  "access_token": "eyJraWQ...",
  "expires_in": 3600,
  "token_type": "Bearer"
}
```

### 8.3 Test the API

```bash
curl -H "Authorization: Bearer YOUR_ACCESS_TOKEN" https://YOUR_API_ENDPOINT/service
```

You should get a JSON response with service information.

### Available OAuth Scopes

| Scope | Access Level |
|-------|-------------|
| `tams-api/admin` | Full administrative access |
| `tams-api/read` | GET and HEAD methods |
| `tams-api/write` | PUT and POST methods |
| `tams-api/delete` | DELETE methods |

You can combine scopes: `scope=tams-api/read tams-api/write`

---

## Part 9: Adding Additional Storage Backends (Optional)

To add extra S3 buckets as separate media storage backends:

```bash
aws cloudformation create-stack ^
  --stack-name tams-storage-2 ^
  --template-body file://storage_backend.yaml ^
  --parameters ParameterKey=ApiStackName,ParameterValue=tams ^
  --capabilities CAPABILITY_IAM
```

This does not require SAM CLI — it is a standard CloudFormation deployment.

---

## Part 10: Redeploying After Code Changes

Once `samconfig.toml` exists, future deploys are simple:

```bash
sam build --use-container
sam deploy
```

---

## Part 11: Cleanup (Delete Everything)

### Delete the TAMS Stack

```bash
sam delete --stack-name tams
```

### Delete Retained Resources Manually

After `sam delete`, these resources are retained (to protect data) and must be deleted manually via the AWS Console:

1. **S3 Bucket** — Go to S3 Console, empty the bucket, then delete it
2. **DynamoDB Tables** — Go to DynamoDB Console and delete the tables

### Delete Additional Storage Stacks

```bash
aws cloudformation delete-stack --stack-name tams-storage-2
```

### Delete the IAM User (Optional)

If you no longer need the `tams-deployer` user:
1. Go to **IAM** → **Users** → `tams-deployer`
2. Delete access keys first
3. Delete the user

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `sam build` fails | Ensure Docker Desktop is running. Check internet connectivity. |
| `make` not found on Windows | Install via `choco install make` or use WSL/Git Bash |
| Deployment times out | Neptune takes ~20 min. Wait and check CloudFormation console for progress. |
| `CAPABILITY_IAM` error | Re-run `sam deploy --guided` and answer `y` to IAM role creation |
| API returns 403 Forbidden | Verify your token is valid and includes the correct scopes |
| API returns 401 Unauthorized | Check the Authorization header format: `Bearer <token>` |
| Lambda timeout / connectivity | Ensure VPC has proper endpoints. If using existing VPC, verify private subnets. |
| CloudWatch logging error | Complete Part 4 (API Gateway CloudWatch role setup) |
| Docker permission errors | Run Docker Desktop as administrator, or add your user to the `docker-users` group |
| Python version mismatch | This project requires Python 3.14. Check with `python --version`. |

---

## Architecture Reference

```
                         ┌──────────┐
                         │   WAF    │
                         └────┬─────┘
                              │
                    ┌─────────▼──────────┐
                    │   API Gateway       │
                    │  (REST + Lambda     │
                    │   Authorizer)       │
                    └─────────┬──────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
   ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
   │  Sources    │    │   Flows     │    │  Segments   │
   │  Lambda     │    │   Lambda    │    │  Lambda     │
   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
          │                   │                   │
   ┌──────▼───────────────────▼───────────────────▼──────┐
   │                    VPC (Private Subnets)             │
   │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
   │  │ Neptune  │  │ DynamoDB │  │  S3 (Media Store) │  │
   │  │ (Graph)  │  │ (Tables) │  │                   │  │
   │  └──────────┘  └──────────┘  └──────────────────┘  │
   │                                                      │
   │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
   │  │   SQS    │  │EventBridge│  │    Cognito       │  │
   │  │ (Queues) │  │ (Events) │  │  (Auth)          │  │
   │  └──────────┘  └──────────┘  └──────────────────┘  │
   └──────────────────────────────────────────────────────┘
```
