# TAMS API — Complete Testing Guide (All Endpoints)

All commands use `curl` on Ubuntu. Run them in order — later steps depend on earlier ones.

---

## Setup: Environment Variables

Run this once to set up all variables:

```bash
REGION="ap-south-1"
STACK="tams"
COGNITO_HOST="674904383813-tams-06bde0395d31.auth.ap-south-2.amazoncognito.com"
RESOLVE="--resolve ${COGNITO_HOST}:443:18.61.249.73"

API_URL=$(aws cloudformation describe-stacks --stack-name $STACK --region $REGION \
  --query "Stacks[0].Outputs[?OutputKey=='ApiEndpoint'].OutputValue" --output text)

USER_POOL_ID=$(aws cloudformation describe-stacks --stack-name $STACK --region $REGION \
  --query "Stacks[0].Outputs[?OutputKey=='UserPoolId'].OutputValue" --output text)

CLIENT_ID=$(aws cloudformation describe-stacks --stack-name $STACK --region $REGION \
  --query "Stacks[0].Outputs[?OutputKey=='UserPoolClientId'].OutputValue" --output text)

TOKEN_URL=$(aws cloudformation describe-stacks --stack-name $STACK --region $REGION \
  --query "Stacks[0].Outputs[?OutputKey=='TokenUrl'].OutputValue" --output text)

CLIENT_SECRET=$(aws cognito-idp describe-user-pool-client \
  --user-pool-id $USER_POOL_ID --client-id $CLIENT_ID --region $REGION \
  --query "UserPoolClient.ClientSecret" --output text)

echo "API_URL:       $API_URL"
echo "CLIENT_ID:     $CLIENT_ID"
echo "CLIENT_SECRET: ${CLIENT_SECRET:0:10}..."
echo "TOKEN_URL:     $TOKEN_URL"
```

---

## Setup: Get Tokens

### Read + Write Token

```bash
TOKEN=$(curl -s -X POST "$TOKEN_URL" $RESOLVE \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=$CLIENT_ID&client_secret=$CLIENT_SECRET&grant_type=client_credentials&scope=tams-api/read tams-api/write" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
echo "TOKEN set: ${TOKEN:0:20}..."
```

### Admin Token (all scopes)

```bash
ADMIN_TOKEN=$(curl -s -X POST "$TOKEN_URL" $RESOLVE \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=$CLIENT_ID&client_secret=$CLIENT_SECRET&grant_type=client_credentials&scope=tams-api/admin tams-api/read tams-api/write tams-api/delete" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
echo "ADMIN_TOKEN set: ${ADMIN_TOKEN:0:20}..."
```

### Helper alias

```bash
AUTH="-H \"Authorization: Bearer $TOKEN\""
```

---

## Setup: Generate UUIDs

```bash
SOURCE_ID=$(python3 -c "import uuid; print(uuid.uuid4())")
FLOW_ID=$(python3 -c "import uuid; print(uuid.uuid4())")
FLOW_ID_AUDIO=$(python3 -c "import uuid; print(uuid.uuid4())")
SOURCE_ID_AUDIO=$(python3 -c "import uuid; print(uuid.uuid4())")
FLOW_ID_DATA=$(python3 -c "import uuid; print(uuid.uuid4())")
SOURCE_ID_DATA=$(python3 -c "import uuid; print(uuid.uuid4())")

echo "SOURCE_ID:       $SOURCE_ID"
echo "FLOW_ID (video): $FLOW_ID"
echo "FLOW_ID (audio): $FLOW_ID_AUDIO"
echo "FLOW_ID (data):  $FLOW_ID_DATA"
```

---

## 1. Root Endpoint

### GET /

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/" | python3 -m json.tool
```

### HEAD /

```bash
curl -s -I -H "Authorization: Bearer $TOKEN" "$API_URL/"
```

---

## 2. Service

### GET /service

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/service" | python3 -m json.tool
```

### HEAD /service

```bash
curl -s -I -H "Authorization: Bearer $TOKEN" "$API_URL/service"
```

### POST /service (admin only)

```bash
curl -s -X POST "$API_URL/service" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type": "urn:x-tams:service.example"}' | python3 -m json.tool
```

---

## 3. Storage Backends

### GET /service/storage-backends

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/service/storage-backends" | python3 -m json.tool
```

### HEAD /service/storage-backends

```bash
curl -s -I -H "Authorization: Bearer $TOKEN" "$API_URL/service/storage-backends"
```

---

## 4. Webhooks

### POST /service/webhooks — Create a Webhook

```bash
curl -s -X POST "$API_URL/service/webhooks" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://httpbin.org/post",
    "events": ["flows/created", "flows/updated", "flows/deleted", "flows/segments_added", "flows/segments_deleted", "sources/created", "sources/updated", "sources/deleted"]
  }' | python3 -m json.tool
```

Save the webhook ID:

```bash
WEBHOOK_ID="<paste-id-from-response>"
```

### GET /service/webhooks — List Webhooks

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/service/webhooks" | python3 -m json.tool
```

### HEAD /service/webhooks

```bash
curl -s -I -H "Authorization: Bearer $TOKEN" "$API_URL/service/webhooks"
```

### GET /service/webhooks/{webhookId}

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/service/webhooks/$WEBHOOK_ID" | python3 -m json.tool
```

### PUT /service/webhooks/{webhookId} — Update Webhook

```bash
curl -s -X PUT "$API_URL/service/webhooks/$WEBHOOK_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"id\": \"$WEBHOOK_ID\",
    \"url\": \"https://httpbin.org/post\",
    \"events\": [\"flows/created\", \"flows/updated\"],
    \"status\": \"disabled\"
  }" | python3 -m json.tool
```

### DELETE /service/webhooks/{webhookId}

```bash
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" "$API_URL/service/webhooks/$WEBHOOK_ID" | python3 -m json.tool
```

---

## 5. Create Flows (Video, Audio, Data)

### PUT /flows/{flowId} — Create Video Flow

```bash
curl -s -X PUT "$API_URL/flows/$FLOW_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"id\": \"$FLOW_ID\",
    \"source_id\": \"$SOURCE_ID\",
    \"label\": \"Test Video Flow\",
    \"description\": \"1080p25 H.264 test flow\",
    \"format\": \"urn:x-nmos:format:video\",
    \"codec\": \"video/h264\",
    \"container\": \"video/mp4\",
    \"essence_parameters\": {
      \"frame_width\": 1920,
      \"frame_height\": 1080,
      \"frame_rate\": { \"numerator\": 25, \"denominator\": 1 }
    }
  }" | python3 -m json.tool
```

### PUT /flows/{flowId} — Create Audio Flow

```bash
curl -s -X PUT "$API_URL/flows/$FLOW_ID_AUDIO" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"id\": \"$FLOW_ID_AUDIO\",
    \"source_id\": \"$SOURCE_ID_AUDIO\",
    \"label\": \"Test Audio Flow\",
    \"description\": \"48kHz stereo AAC test flow\",
    \"format\": \"urn:x-nmos:format:audio\",
    \"codec\": \"audio/aac\",
    \"container\": \"audio/mp4\",
    \"essence_parameters\": {
      \"sample_rate\": 48000,
      \"channels\": 2
    }
  }" | python3 -m json.tool
```

### PUT /flows/{flowId} — Create Data Flow

```bash
curl -s -X PUT "$API_URL/flows/$FLOW_ID_DATA" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"id\": \"$FLOW_ID_DATA\",
    \"source_id\": \"$SOURCE_ID_DATA\",
    \"label\": \"Test Data Flow\",
    \"description\": \"JSON metadata flow\",
    \"format\": \"urn:x-nmos:format:data\",
    \"codec\": \"application/json\",
    \"essence_parameters\": {
      \"data_type\": \"urn:x-tams:data:test\"
    }
  }" | python3 -m json.tool
```

---

## 6. Flows — Read Operations

### GET /flows — List All Flows

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows" | python3 -m json.tool
```

### HEAD /flows

```bash
curl -s -I -H "Authorization: Bearer $TOKEN" "$API_URL/flows"
```

### GET /flows?format= — Filter by Format

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows?format=urn:x-nmos:format:video" | python3 -m json.tool
```

### GET /flows/{flowId} — Get Single Flow

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID" | python3 -m json.tool
```

### HEAD /flows/{flowId}

```bash
curl -s -I -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID"
```

---

## 7. Flow Tags

### PUT /flows/{flowId}/tags/{name} — Create Tag

```bash
curl -s -X PUT "$API_URL/flows/$FLOW_ID/tags/environment" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '"testing"' | python3 -m json.tool
```

```bash
curl -s -X PUT "$API_URL/flows/$FLOW_ID/tags/resolution" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '"1080p"' | python3 -m json.tool
```

### GET /flows/{flowId}/tags — List All Tags

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/tags" | python3 -m json.tool
```

### GET /flows/{flowId}/tags/{name} — Get Single Tag

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/tags/environment" | python3 -m json.tool
```

### DELETE /flows/{flowId}/tags/{name} — Delete Tag

```bash
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/tags/resolution"
```

---

## 8. Flow Label

### GET /flows/{flowId}/label

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/label" | python3 -m json.tool
```

### PUT /flows/{flowId}/label

```bash
curl -s -X PUT "$API_URL/flows/$FLOW_ID/label" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '"Updated Video Flow Label"'
```

### DELETE /flows/{flowId}/label

```bash
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/label"
```

---

## 9. Flow Description

### GET /flows/{flowId}/description

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/description" | python3 -m json.tool
```

### PUT /flows/{flowId}/description

```bash
curl -s -X PUT "$API_URL/flows/$FLOW_ID/description" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '"Updated description for the test video flow"'
```

### DELETE /flows/{flowId}/description

```bash
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/description"
```

---

## 10. Flow Read Only

### GET /flows/{flowId}/read_only

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/read_only" | python3 -m json.tool
```

### PUT /flows/{flowId}/read_only — Set Read Only

```bash
curl -s -X PUT "$API_URL/flows/$FLOW_ID/read_only" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d 'true'
```

### PUT /flows/{flowId}/read_only — Unset Read Only

```bash
curl -s -X PUT "$API_URL/flows/$FLOW_ID/read_only" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d 'false'
```

---

## 11. Flow Bit Rates

### PUT /flows/{flowId}/max_bit_rate

```bash
curl -s -X PUT "$API_URL/flows/$FLOW_ID/max_bit_rate" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '50000'
```

### GET /flows/{flowId}/max_bit_rate

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/max_bit_rate" | python3 -m json.tool
```

### DELETE /flows/{flowId}/max_bit_rate

```bash
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/max_bit_rate"
```

### PUT /flows/{flowId}/avg_bit_rate

```bash
curl -s -X PUT "$API_URL/flows/$FLOW_ID/avg_bit_rate" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '25000'
```

### GET /flows/{flowId}/avg_bit_rate

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/avg_bit_rate" | python3 -m json.tool
```

### DELETE /flows/{flowId}/avg_bit_rate

```bash
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/avg_bit_rate"
```

---

## 12. Flow Collection

### PUT /flows/{flowId}/flow_collection — Group Flows Together

```bash
curl -s -X PUT "$API_URL/flows/$FLOW_ID/flow_collection" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "[
    {\"flow_id\": \"$FLOW_ID_AUDIO\", \"role\": \"audio\"}
  ]" | python3 -m json.tool
```

### GET /flows/{flowId}/flow_collection

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/flow_collection" | python3 -m json.tool
```

### DELETE /flows/{flowId}/flow_collection

```bash
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/flow_collection"
```

---

## 13. Sources

### GET /sources — List All Sources

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/sources" | python3 -m json.tool
```

### HEAD /sources

```bash
curl -s -I -H "Authorization: Bearer $TOKEN" "$API_URL/sources"
```

### GET /sources/{sourceId} — Get Single Source

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/sources/$SOURCE_ID" | python3 -m json.tool
```

### HEAD /sources/{sourceId}

```bash
curl -s -I -H "Authorization: Bearer $TOKEN" "$API_URL/sources/$SOURCE_ID"
```

---

## 14. Source Tags

### PUT /sources/{sourceId}/tags/{name}

```bash
curl -s -X PUT "$API_URL/sources/$SOURCE_ID/tags/camera" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '"cam-01"' | python3 -m json.tool
```

### GET /sources/{sourceId}/tags

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/sources/$SOURCE_ID/tags" | python3 -m json.tool
```

### GET /sources/{sourceId}/tags/{name}

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/sources/$SOURCE_ID/tags/camera" | python3 -m json.tool
```

### DELETE /sources/{sourceId}/tags/{name}

```bash
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" "$API_URL/sources/$SOURCE_ID/tags/camera"
```

---

## 15. Source Label

### PUT /sources/{sourceId}/label

```bash
curl -s -X PUT "$API_URL/sources/$SOURCE_ID/label" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '"Camera 1 Source"'
```

### GET /sources/{sourceId}/label

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/sources/$SOURCE_ID/label" | python3 -m json.tool
```

### DELETE /sources/{sourceId}/label

```bash
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" "$API_URL/sources/$SOURCE_ID/label"
```

---

## 16. Source Description

### PUT /sources/{sourceId}/description

```bash
curl -s -X PUT "$API_URL/sources/$SOURCE_ID/description" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '"Main camera source for studio A"'
```

### GET /sources/{sourceId}/description

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/sources/$SOURCE_ID/description" | python3 -m json.tool
```

### DELETE /sources/{sourceId}/description

```bash
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" "$API_URL/sources/$SOURCE_ID/description"
```

---

## 17. Flow Storage

### POST /flows/{flowId}/storage — Register Storage

```bash
curl -s -X POST "$API_URL/flows/$FLOW_ID/storage" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}' | python3 -m json.tool
```

Save an object_id from the response:

```bash
OBJECT_ID="<paste-object_id-from-response>"
```

---

## 18. Flow Segments

### POST /flows/{flowId}/segments — Create Segment

```bash
curl -s -X POST "$API_URL/flows/$FLOW_ID/segments" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"object_id\": \"$OBJECT_ID\",
    \"timerange\": \"0:0_10:0\"
  }" | python3 -m json.tool
```

### GET /flows/{flowId}/segments — List Segments

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/segments" | python3 -m json.tool
```

### HEAD /flows/{flowId}/segments

```bash
curl -s -I -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/segments"
```

### GET /flows/{flowId}/segments?get_urls=true — With Download URLs

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/segments?get_urls=true" | python3 -m json.tool
```

### GET /flows/{flowId}/segments?timerange= — Filter by Timerange

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/$FLOW_ID/segments?timerange=0:0_5:0" | python3 -m json.tool
```

---

## 19. Objects

### GET /objects/{objectId} — Get Object Info

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/objects/$OBJECT_ID" | python3 -m json.tool
```

### HEAD /objects/{objectId}

```bash
curl -s -I -H "Authorization: Bearer $TOKEN" "$API_URL/objects/$OBJECT_ID"
```

### POST /objects/{objectId}/instances — Register Object Instance (External URL)

```bash
curl -s -X POST "$API_URL/objects/$OBJECT_ID/instances" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/media/test-segment.mp4",
    "label": "external-backup"
  }' | python3 -m json.tool
```

### DELETE /objects/{objectId}/instances — Remove Object Instance

```bash
curl -s -X DELETE "$API_URL/objects/$OBJECT_ID/instances" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/media/test-segment.mp4",
    "label": "external-backup"
  }' | python3 -m json.tool
```

---

## 20. Delete Flow Segments (delete scope)

### DELETE /flows/{flowId}/segments — Delete Segments by Timerange

```bash
curl -s -X DELETE "$API_URL/flows/$FLOW_ID/segments?timerange=0:0_10:0" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | python3 -m json.tool
```

---

## 21. Delete Flow (delete scope)

### DELETE /flows/{flowId}

```bash
curl -s -X DELETE "$API_URL/flows/$FLOW_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | python3 -m json.tool
```

---

## 22. Flow Delete Requests (admin scope)

### GET /flow-delete-requests — List Delete Requests

```bash
curl -s -H "Authorization: Bearer $ADMIN_TOKEN" "$API_URL/flow-delete-requests" | python3 -m json.tool
```

### HEAD /flow-delete-requests

```bash
curl -s -I -H "Authorization: Bearer $ADMIN_TOKEN" "$API_URL/flow-delete-requests"
```

### GET /flow-delete-requests/{request-id} — Get Specific Delete Request

```bash
DELETE_REQUEST_ID="<paste-from-delete-response-or-list>"
curl -s -H "Authorization: Bearer $ADMIN_TOKEN" "$API_URL/flow-delete-requests/$DELETE_REQUEST_ID" | python3 -m json.tool
```

---

## 23. Cleanup — Delete Remaining Test Flows

```bash
# Delete audio flow
curl -s -X DELETE "$API_URL/flows/$FLOW_ID_AUDIO" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | python3 -m json.tool

# Delete data flow
curl -s -X DELETE "$API_URL/flows/$FLOW_ID_DATA" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | python3 -m json.tool
```

### Verify Everything is Cleaned Up

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows" | python3 -m json.tool
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/sources" | python3 -m json.tool
```

---

## 24. Error Testing

### 401 — No Token

```bash
curl -s "$API_URL/service" | python3 -m json.tool
```

### 403 — Wrong Scope (read token on delete endpoint)

```bash
curl -s -X DELETE "$API_URL/flows/$FLOW_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

### 404 — Non-existent Resource

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/flows/00000000-0000-0000-0000-000000000000" | python3 -m json.tool
```

### 400 — Invalid Request Body

```bash
curl -s -X PUT "$API_URL/flows/not-a-uuid" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"invalid": "data"}' | python3 -m json.tool
```

---

## Quick Reference — All Endpoints

| # | Method | Endpoint | Scope |
|---|--------|----------|-------|
| 1 | GET/HEAD | `/` | read |
| 2 | GET/HEAD | `/service` | read |
| 3 | POST | `/service` | admin |
| 4 | GET/HEAD | `/service/storage-backends` | read |
| 5 | GET/HEAD/POST | `/service/webhooks` | read/write |
| 6 | GET/HEAD/PUT/DELETE | `/service/webhooks/{id}` | read |
| 7 | GET/HEAD | `/sources` | read |
| 8 | GET/HEAD | `/sources/{id}` | read |
| 9 | GET/HEAD | `/sources/{id}/tags` | read |
| 10 | GET/PUT/DELETE | `/sources/{id}/tags/{name}` | read/write |
| 11 | GET/PUT/DELETE | `/sources/{id}/label` | read/write |
| 12 | GET/PUT/DELETE | `/sources/{id}/description` | read/write |
| 13 | GET/HEAD | `/flows` | read |
| 14 | GET/HEAD/PUT | `/flows/{id}` | read/write |
| 15 | DELETE | `/flows/{id}` | delete |
| 16 | GET/HEAD | `/flows/{id}/tags` | read |
| 17 | GET/PUT/DELETE | `/flows/{id}/tags/{name}` | read/write |
| 18 | GET/PUT/DELETE | `/flows/{id}/label` | read/write |
| 19 | GET/PUT/DELETE | `/flows/{id}/description` | read/write |
| 20 | GET/PUT | `/flows/{id}/read_only` | read/write |
| 21 | GET/PUT/DELETE | `/flows/{id}/flow_collection` | read/write |
| 22 | GET/PUT/DELETE | `/flows/{id}/max_bit_rate` | read/write |
| 23 | GET/PUT/DELETE | `/flows/{id}/avg_bit_rate` | read/write |
| 24 | POST | `/flows/{id}/storage` | write |
| 25 | GET/HEAD/POST | `/flows/{id}/segments` | read/write |
| 26 | DELETE | `/flows/{id}/segments` | delete |
| 27 | GET/HEAD | `/objects/{id}` | read |
| 28 | POST/DELETE | `/objects/{id}/instances` | write |
| 29 | GET/HEAD | `/flow-delete-requests` | admin |
| 30 | GET/HEAD | `/flow-delete-requests/{id}` | admin/delete |

## OAuth Scopes

| Scope | Access |
|-------|--------|
| `tams-api/admin` | Full access to everything |
| `tams-api/read` | GET and HEAD on all endpoints |
| `tams-api/write` | PUT and POST operations |
| `tams-api/delete` | DELETE operations |

## Webhook Event Types

| Event | Trigger |
|-------|---------|
| `flows/created` | New flow created |
| `flows/updated` | Flow metadata updated |
| `flows/deleted` | Flow deleted |
| `flows/segments_added` | Segments added to a flow |
| `flows/segments_deleted` | Segments deleted from a flow |
| `sources/created` | New source created |
| `sources/updated` | Source metadata updated |
| `sources/deleted` | Source deleted |
