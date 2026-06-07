# TAMS Deployment Guide — Ubuntu (WSL2)

Run all commands inside your WSL2 Ubuntu terminal.

---

## Part 1: Install Required Tools

### 1.1 Update System

```bash
sudo apt update && sudo apt upgrade -y
```

### 1.2 Install Python 3.14

```bash
sudo apt install -y software-properties-common
sudo add-apt-repository -y ppa:deadsnakes/ppa
sudo apt update
sudo apt install -y python3.14 python3.14-venv python3.14-dev
```

Verify:

```bash
python3.14 --version
```

### 1.3 Install pip for Python 3.14

```bash
curl -sS https://bootstrap.pypa.io/get-pip.py | python3.14
```

### 1.4 Install Poetry

```bash
python3.14 -m pip install poetry --ignore-installed urllib3
```

> The `--ignore-installed urllib3` flag is needed because Ubuntu's system urllib3 (installed via apt) conflicts with pip. This bypasses the conflict.

Verify:

```bash
poetry --version
```

### 1.5 Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install -y unzip
unzip awscliv2.zip
sudo ./aws/install
rm -rf aws awscliv2.zip
```

Verify:

```bash
aws --version
```

### 1.6 Install AWS SAM CLI

```bash
python3.14 -m pip install aws-sam-cli --ignore-installed urllib3
```

Verify:

```bash
sam --version
```

### 1.7 Install Docker

```bash
# Add Docker's official GPG key and repo
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Start Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add your user so you don't need sudo for docker commands
sudo usermod -aG docker $USER
```

Verify:

```bash
docker --version
```

### 1.8 Install Git

```bash
sudo apt install -y git
git --version
```

### 1.9 Install Make

```bash
sudo apt install -y make
```

---

## Part 2: Configure AWS CLI

```bash
aws configure
```

| Prompt | Value |
|--------|-------|
| AWS Access Key ID | Your `tams-deployer` access key |
| AWS Secret Access Key | Your `tams-deployer` secret key |
| Default region name | `us-east-1` |
| Default output format | `json` |

Verify:

```bash
aws sts get-caller-identity
```

---

## Part 3: Configure API Gateway CloudWatch Logging (One-Time Per Region)

### 3.1 Create the IAM Role

```bash
cat > apigw-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "apigateway.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name APIGatewayCloudWatchRole \
  --assume-role-policy-document file://apigw-trust-policy.json
```

### 3.2 Attach the CloudWatch Policy

```bash
aws iam attach-role-policy \
  --role-name APIGatewayCloudWatchRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonAPIGatewayPushToCloudWatchLogs
```

### 3.3 Assign the Role to API Gateway

```bash
ROLE_ARN=$(aws iam get-role --role-name APIGatewayCloudWatchRole --query "Role.Arn" --output text)

aws apigateway update-account \
  --patch-operations "op=replace,path=/cloudwatchRoleArn,value=$ROLE_ARN"
```

### 3.4 Verify

```bash
aws apigateway get-account
```

### 3.5 Cleanup

```bash
rm apigw-trust-policy.json
```

---

## Part 4: Clone and Prepare the Project

```bash
cd /mnt/d/skandha
git clone https://github.com/awslabs/time-addressable-media-store.git
cd time-addressable-media-store
git checkout v6.1.0
```

### 4.1 Set Poetry to Use Python 3.14

```bash
poetry env use python3.14
```

### 4.2 Install Dependencies

```bash
poetry install
```

### 4.3 Generate API Spec

```bash
make api-spec
```

---

## Part 5: Build

```bash
sam build --use-container
```

This pulls a Docker container matching the Lambda runtime and builds all functions. First run takes 5-10 minutes.

---

## Part 6: Deploy

```bash
sam deploy --guided
```

### Deployment Parameters (From Scratch)

| Prompt | Value |
|--------|-------|
| Stack Name | `tams` |
| AWS Region | `us-east-1` |
| VpcId | *(press Enter — blank)* |
| VpcAZs | *(press Enter — blank)* |
| PrivateSubnetIds | *(press Enter — blank)* |
| NeptuneDBInstanceClass | `db.serverless` |
| NeptuneServerlessConfiguration | `1,128` |
| DeployWaf | `Yes` |
| JwtIssuerUrl | *(press Enter — blank)* |
| LambdaAuthorizerArn | *(press Enter — blank)* |
| Confirm changes before deploy | `y` |
| Allow SAM CLI IAM role creation | `y` |
| Disable rollback | `n` |
| Save arguments to samconfig.toml | `y` |

Expect 25-35 minutes. Neptune alone takes ~20 minutes.

---

## Part 7: Get API Credentials

### 7.1 Get Stack Outputs

```bash
aws cloudformation describe-stacks --stack-name tams --query "Stacks[0].Outputs" --output table
```

### 7.2 Get Cognito Client Secret

```bash
USER_POOL_ID=$(aws cloudformation describe-stacks --stack-name tams --query "Stacks[0].Outputs[?OutputKey=='UserPoolId'].OutputValue" --output text)

CLIENT_ID=$(aws cloudformation describe-stacks --stack-name tams --query "Stacks[0].Outputs[?OutputKey=='UserPoolClientId'].OutputValue" --output text)

aws cognito-idp describe-user-pool-client \
  --user-pool-id $USER_POOL_ID \
  --client-id $CLIENT_ID \
  --query "UserPoolClient.ClientSecret" \
  --output text
```

### 7.3 Get Access Token

```bash
TOKEN_URL=$(aws cloudformation describe-stacks --stack-name tams --query "Stacks[0].Outputs[?OutputKey=='TokenUrl'].OutputValue" --output text)

CLIENT_SECRET="<paste-secret-from-above>"

curl -s -X POST "$TOKEN_URL" \
  -d "client_id=$CLIENT_ID&client_secret=$CLIENT_SECRET&grant_type=client_credentials&scope=tams-api/read tams-api/write"
```

### 7.4 Test the API

```bash
API_URL=$(aws cloudformation describe-stacks --stack-name tams --query "Stacks[0].Outputs[?OutputKey=='ApiEndpoint'].OutputValue" --output text)

ACCESS_TOKEN="<paste-token-from-above>"

curl -H "Authorization: Bearer $ACCESS_TOKEN" "$API_URL/service"
```

---

## Part 8: Cleanup

```bash
sam delete --stack-name tams
```

Then manually delete retained S3 bucket and DynamoDB tables from the AWS Console.

---

## Part 9: Redeploy After Changes

```bash
sam build --use-container
sam deploy
```
