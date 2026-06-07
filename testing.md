# TAMS API — Testing & User Guide

This guide walks you through testing the deployed TAMS stack step by step.

---

## Step 1: Get Your Stack Outputs

```bash
REGION="ap-south-2"
STACK="tams"

aws cloudformation describe-stacks \
  --stack-name $STACK \
  --region $REGION \
  --query "Stacks[0].Outputs" \
  --output table
```

Save these values as environment variables for easy use:

```bash
API_URL=$(aws cloudformation describe-stacks --stack-name $STACK --region $REGION \
  --query "Stacks[0].Outputs[?OutputKey=='ApiEndpoint'].OutputValue" --output text)

USER_POOL_ID=$(aws cloudformation describe-stacks --stack-name $STACK --region $REGION \
  --query "Stacks[0].Outputs[?OutputKey=='UserPoolId'].OutputValue" --output text)

CLIENT_ID=$(aws cloudformation describe-stacks --stack-name $STACK --region $REGION \
  --query "Stacks[0].Outputs[?OutputKey=='UserPoolClientId'].OutputValue" --output text)

TOKEN_URL=$(aws cloudformation describe-stacks --stack-name $STACK --region $REGION \
  --query "Stacks[0].Outputs[?OutputKey=='TokenUrl'].OutputValue" --output text)

echo "API_URL:      $API_URL"
echo "USER_POOL_ID: $USER_POOL_ID"
echo "CLIENT_ID:    $CLIENT_ID"
echo "TOKEN_URL:    $TOKEN_URL"
```

---

## Step 2: Get the Cognito Client Secret

```bash
CLIENT_SECRET=$(aws cognito-idp describe-user-pool-client \
  --user-pool-id $USER_POOL_ID \
  --client-id $CLIENT_ID \
  --region $REGION \
  --query "UserPoolClient.ClientSecret" \
  --output text)

echo "CLIENT_SECRET: $CLIENT_SECRET"
```

---

## Step 3: Get an Access Token

### 3.1 Token with Read + Write access

```bash
TOKEN=$(curl -s -X POST "$TOKEN_URL" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=$CLIENT_ID&client_secret=$CLIENT_SECRET&grant_type=client_credentials&scope=tams-api/read tams-api/write" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

echo "TOKEN: ${TOKEN:0:50}..."
```

### 3.2 Token with All access (admin + read + write + delete)

```bash
ADMIN_TOKEN=$(curl -s -X POST "$TOKEN_URL" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=$CLIENT_ID&client_secret=$CLIENT_SECRET&grant_type=client_credentials&scope=tams-api/admin tams-api/read tams-api/write tams-api/delete" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

echo "ADMIN_TOKEN: ${ADMIN_TOKEN:0:50}..."
```

---

## Step 4: Test the Service Endpoint

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/service" | python3 -m json.tool
```

Expected: JSON with service information.

---

## Step 5: Check Storage Backends

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/service/storage-backends" | python3 -m json.tool
```

Expected: List of available storage backends (at least one S3 bucket).

---

## Step 6: List Sources (Initially Empty)

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/sources" | python3 -m json.tool
```

Expected: Empty list `[]` or `{"results": []}`.

---

## Step 7: List Flows (Initially Empty)

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows" | python3 -m json.tool
```

Expected: Empty list.

---

## Step 8: Create a Flow

A flow represents a time-based media stream. You need a source ID and flow ID (UUIDs).

```bash
SOURCE_ID=$(python3 -c "import uuid; print(uuid.uuid4())")
FLOW_ID=$(python3 -c "import uuid; print(uuid.uuid4())")

echo "SOURCE_ID: $SOURCE_ID"
echo "FLOW_ID:   $FLOW_ID"
```

Create the flow:

```bash
curl -s -X PUT "$API_URL/flows/$FLOW_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"id\": \"$FLOW_ID\",
    \"source_id\": \"$SOURCE_ID\",
    \"label\": \"Test Flow\",
    \"description\": \"My first 2nc flow\",
    \"format\": \"urn:x-nmos:format:video\",
    \"codec\": \"video/h264\",
    \"essence_parameters\": {
      \"frame_width\": 1920,
      \"frame_height\": 1080,
      \"frame_rate\": { \"numerator\": 25, \"denominator\": 1 }
    }
  }" | python3 -m json.tool
```

Expected: JSON with the created flow details.

---

## Step 9: Verify the Flow Was Created

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID" | python3 -m json.tool
```

---

## Step 10: Verify the Source Was Auto-Created

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/sources/$SOURCE_ID" | python3 -m json.tool
```

---

## Step 11: Add Tags to a Flow

```bash
curl -s -X PUT "$API_URL/flows/$FLOW_ID/tags/environment" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '"testing"' | python3 -m json.tool
```

Read the tag back:

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/tags/environment" | python3 -m json.tool
```

---

## Step 12: Set Flow Label and Description

```bash
# Update label
curl -s -X PUT "$API_URL/flows/$FLOW_ID/label" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '"Updated Test Flow"'

# Update description
curl -s -X PUT "$API_URL/flows/$FLOW_ID/description" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '"This flow was updated via the API"'
```

---

## Step 13: Register Flow Storage

Before uploading segments, register storage for the flow:

```bash
curl -s -X POST "$API_URL/flows/$FLOW_ID/storage" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}' | python3 -m json.tool
```

Expected: JSON with storage details including the S3 bucket.

---

## Step 14: Create Flow Segments

Segments represent time ranges of media content in a flow.

```bash
curl -s -X POST "$API_URL/flows/$FLOW_ID/segments" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "timerange": "0:0_10:0",
    "get_urls": true
  }' | python3 -m json.tool
```

Expected: JSON with segment details and pre-signed S3 URLs for uploading media.

---

## Step 15: List Flow Segments

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/segments" | python3 -m json.tool
```

---

## Step 16: Set Up a Webhook (Optional)

Webhooks notify external services when changes happen.

```bash
curl -s -X POST "$API_URL/service/webhooks" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "label": "Test Webhook",
    "url": "https://httpbin.org/post",
    "events": ["flow_created", "flow_updated", "segments_updated"]
  }' | python3 -m json.tool
```

List webhooks:

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/service/webhooks" | python3 -m json.tool
```

---

## Step 17: Delete a Flow (Requires Delete Scope)

```bash
curl -s -X DELETE "$API_URL/flows/$FLOW_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | python3 -m json.tool
```

Check delete requests:

```bash
curl -s -H "Authorization: Bearer $ADMIN_TOKEN" "$API_URL/flow-delete-requests" | python3 -m json.tool
```

---

## Step 18: Verify Cleanup

After deletion, verify the flow is gone:

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID"
```

Expected: 404 Not Found.

---

## Quick Reference — All API Endpoints

| Method | Endpoint | Scope Required |
|--------|----------|---------------|
| GET | `/service` | read |
| GET | `/service/storage-backends` | read |
| GET/POST | `/service/webhooks` | read / write |
| GET/PUT/DELETE | `/service/webhooks/{id}` | read |
| GET | `/sources` | read |
| GET | `/sources/{id}` | read |
| GET/PUT/DELETE | `/sources/{id}/tags/{name}` | read / write |
| GET/PUT/DELETE | `/sources/{id}/label` | read / write |
| GET/PUT/DELETE | `/sources/{id}/description` | read / write |
| GET | `/flows` | read |
| GET/PUT | `/flows/{id}` | read / write |
| DELETE | `/flows/{id}` | delete |
| GET/PUT/DELETE | `/flows/{id}/tags/{name}` | read / write |
| GET/PUT/DELETE | `/flows/{id}/label` | read / write |
| GET/PUT/DELETE | `/flows/{id}/description` | read / write |
| POST | `/flows/{id}/storage` | write |
| GET/POST | `/flows/{id}/segments` | read / write |
| DELETE | `/flows/{id}/segments` | delete |
| GET | `/objects/{id}` | read |
| GET | `/flow-delete-requests` | admin |
| GET | `/flow-delete-requests/{id}` | admin / delete |

## OAuth Scopes

| Scope | Access |
|-------|--------|
| `tams-api/admin` | Full access to everything |
| `tams-api/read` | GET and HEAD on all endpoints |
| `tams-api/write` | PUT and POST operations |
| `tams-api/delete` | DELETE operations |

Combine scopes as needed: `scope=tams-api/read tams-api/write tams-api/delete`

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `401 Unauthorized` | Token expired (tokens last 1 hour). Get a new one from Step 3. |
| `403 Forbidden` | Token doesn't have the required scope. Check which scope the endpoint needs. |
| `404 Not Found` | Resource doesn't exist or wrong URL. Check the API_URL and resource ID. |
| `500 Internal Server Error` | Check CloudWatch Logs for the Lambda function in the AWS Console. |
| Token request fails | Verify CLIENT_ID, CLIENT_SECRET, and TOKEN_URL are correct. |
