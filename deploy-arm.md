# ARM Template Deployment Guide
## HR Assistant WITH Responsible AI

This guide walks you through deploying all 3 Azure services using an ARM template.

**Time Required:** ~5 minutes (automated) + ~5 minutes (manual steps)  
**Prerequisites:** Azure CLI installed OR Cloud Shell access

---

## 🚀 Quick Deployment

### Option A: Azure CLI (Local)

1. **Login to Azure**
   ```bash
   az login
   ```

2. **Set Subscription** (if multiple)
   ```bash
   az account list --output table
   az account set --subscription "Your Subscription Name"
   ```

3. **Create Resource Group**
   ```bash
   az group create \
     --name rg-hr-assistant-rai \
     --location eastus
   ```

4. **Deploy ARM Template**
   ```bash
   az deployment group create \
     --resource-group rg-hr-assistant-rai \
     --template-file deploy-arm.json \
     --parameters searchServiceSku=basic openAIModelName=gpt-4o
   ```

   Wait ~3-4 minutes for deployment.

5. **View Outputs**
   ```bash
   az deployment group show \
     --resource-group rg-hr-assistant-rai \
     --name deploy-arm \
     --query properties.outputs
   ```

   Copy all endpoint values.

✅ **Services Deployed!** But not done yet - continue below.

---

### Option B: Azure Cloud Shell

1. **Open Cloud Shell**
   - Go to portal.azure.com
   - Click Cloud Shell icon (>_)
   - Select "Bash"

2. **Upload ARM Template**
   - Click "Upload/Download files" icon
   - Upload `deploy-arm.json`

3. **Create Resource Group**
   ```bash
   az group create \
     --name rg-hr-assistant-rai \
     --location eastus
   ```

4. **Deploy Template**
   ```bash
   az deployment group create \
     --resource-group rg-hr-assistant-rai \
     --template-file deploy-arm.json
   ```

5. **View Outputs**
   ```bash
   az deployment group show \
     --resource-group rg-hr-assistant-rai \
     --name deploy-arm \
     --query properties.outputs
   ```

---

## 🔧 Post-Deployment Steps (Required)

The ARM template creates the services, but you need to manually:
1. Get API keys
2. Create search index
3. Upload policy documents

### Step 1: Get API Keys

**Azure OpenAI Key:**
```bash
az cognitiveservices account keys list \
  --resource-group rg-hr-assistant-rai \
  --name $(az deployment group show \
    --resource-group rg-hr-assistant-rai \
    --name deploy-arm \
    --query properties.outputs.openAIName.value -o tsv)
```

**Azure AI Search Key:**
```bash
az search admin-key show \
  --resource-group rg-hr-assistant-rai \
  --service-name $(az deployment group show \
    --resource-group rg-hr-assistant-rai \
    --name deploy-arm \
    --query properties.outputs.searchName.value -o tsv)
```

**Content Safety Key:**
```bash
az cognitiveservices account keys list \
  --resource-group rg-hr-assistant-rai \
  --name $(az deployment group show \
    --resource-group rg-hr-assistant-rai \
    --name deploy-arm \
    --query properties.outputs.contentSafetyName.value -o tsv)
```

**Or get from Portal:**
- Each service → "Keys and Endpoint" → Copy Key 1

---

### Step 2: Create Search Index

**Option A: Azure Portal (Easier)**

1. Go to your Search service in Portal
2. Click "Indexes" → "+ Add index"
3. Index name: `hr-policies`
4. Add fields:

   | Field | Type | Key | Searchable | Filterable |
   |-------|------|-----|------------|------------|
   | id | Edm.String | ✅ | ❌ | ❌ |
   | policy_name | Edm.String | ❌ | ✅ | ✅ |
   | category | Edm.String | ❌ | ❌ | ✅ |
   | content | Edm.String | ❌ | ✅ | ❌ |

5. Click "Create"

**Option B: Azure CLI**

```bash
# Create index schema file
cat > index-schema.json << 'EOF'
{
  "name": "hr-policies",
  "fields": [
    {"name": "id", "type": "Edm.String", "key": true, "searchable": false},
    {"name": "policy_name", "type": "Edm.String", "searchable": true, "filterable": true},
    {"name": "category", "type": "Edm.String", "filterable": true},
    {"name": "content", "type": "Edm.String", "searchable": true}
  ]
}
EOF

# Create index
SEARCH_NAME=$(az deployment group show \
  --resource-group rg-hr-assistant-rai \
  --name deploy-arm \
  --query properties.outputs.searchName.value -o tsv)

SEARCH_KEY=$(az search admin-key show \
  --resource-group rg-hr-assistant-rai \
  --service-name $SEARCH_NAME \
  --query primaryKey -o tsv)

curl -X POST "https://${SEARCH_NAME}.search.windows.net/indexes?api-version=2024-07-01" \
  -H "Content-Type: application/json" \
  -H "api-key: ${SEARCH_KEY}" \
  -d @index-schema.json
```

---

### Step 3: Upload Policy Documents

**Option A: Azure Portal (Easier)**

1. Go to Search service → Indexes → hr-policies
2. Click "Upload documents"
3. Select `sample-policies.json` from your repo
4. Click "Upload"
5. Wait ~1 minute for indexing

**Option B: Python Script**

```bash
# Install Azure SDK
pip install azure-search-documents

# Run upload script
python << 'EOF'
import json
from azure.search.documents import SearchClient
from azure.core.credentials import AzureKeyCredential
import os

# Load sample policies
with open('sample-policies.json', 'r') as f:
    policies = json.load(f)

# Get credentials from environment or ARM outputs
endpoint = os.getenv("AZURE_SEARCH_ENDPOINT")
key = os.getenv("AZURE_SEARCH_KEY")

client = SearchClient(
    endpoint=endpoint,
    index_name="hr-policies",
    credential=AzureKeyCredential(key)
)

# Upload documents
result = client.upload_documents(documents=policies)
print(f"✅ Uploaded {len(policies)} policy documents")
EOF
```

**Option C: Azure CLI**

```bash
# Upload documents via REST API
SEARCH_NAME=$(az deployment group show \
  --resource-group rg-hr-assistant-rai \
  --name deploy-arm \
  --query properties.outputs.searchName.value -o tsv)

SEARCH_KEY=$(az search admin-key show \
  --resource-group rg-hr-assistant-rai \
  --service-name $SEARCH_NAME \
  --query primaryKey -o tsv)

curl -X POST "https://${SEARCH_NAME}.search.windows.net/indexes/hr-policies/docs/index?api-version=2024-07-01" \
  -H "Content-Type: application/json" \
  -H "api-key: ${SEARCH_KEY}" \
  -d @sample-policies.json
```

---

### Step 4: Set Environment Variables

**From ARM outputs:**

```bash
# Get all values
OPENAI_ENDPOINT=$(az deployment group show \
  --resource-group rg-hr-assistant-rai \
  --name deploy-arm \
  --query properties.outputs.openAIEndpoint.value -o tsv)

SEARCH_ENDPOINT=$(az deployment group show \
  --resource-group rg-hr-assistant-rai \
  --name deploy-arm \
  --query properties.outputs.searchEndpoint.value -o tsv)

CONTENT_SAFETY_ENDPOINT=$(az deployment group show \
  --resource-group rg-hr-assistant-rai \
  --name deploy-arm \
  --query properties.outputs.contentSafetyEndpoint.value -o tsv)

# Set environment variables
export AZURE_OPENAI_ENDPOINT="$OPENAI_ENDPOINT"
export AZURE_OPENAI_KEY="<key from step 1>"
export AZURE_SEARCH_ENDPOINT="$SEARCH_ENDPOINT"
export AZURE_SEARCH_KEY="<key from step 1>"
export AZURE_SEARCH_INDEX="hr-policies"
export CONTENT_SAFETY_ENDPOINT="$CONTENT_SAFETY_ENDPOINT"
export CONTENT_SAFETY_KEY="<key from step 1>"
```

---

### Step 5: Run Demo

```bash
pip install -r requirements.txt
python hr_assistant.py
```

---

## 🔧 Customization Options

### Deploy with GPT-4o-mini (Cheaper)

```bash
az deployment group create \
  --resource-group rg-hr-assistant-rai \
  --template-file deploy-arm.json \
  --parameters openAIModelName=gpt-4o-mini
```

### Use Free Search Tier

```bash
az deployment group create \
  --resource-group rg-hr-assistant-rai \
  --template-file deploy-arm.json \
  --parameters searchServiceSku=free
```

**Note:** Free tier limits:
- 50MB storage
- 3 indexes max
- Good for testing only

### Custom Resource Names

```bash
az deployment group create \
  --resource-group rg-hr-assistant-rai \
  --template-file deploy-arm.json \
  --parameters resourceBaseName=mycompany-hr-demo
```

---

## 🔍 Verification

### Check Deployment Status

```bash
az deployment group list \
  --resource-group rg-hr-assistant-rai \
  --output table
```

### List All Resources

```bash
az resource list \
  --resource-group rg-hr-assistant-rai \
  --output table
```

### Verify Search Index

```bash
SEARCH_NAME=$(az deployment group show \
  --resource-group rg-hr-assistant-rai \
  --name deploy-arm \
  --query properties.outputs.searchName.value -o tsv)

SEARCH_KEY=$(az search admin-key show \
  --resource-group rg-hr-assistant-rai \
  --service-name $SEARCH_NAME \
  --query primaryKey -o tsv)

# Test search
curl "https://${SEARCH_NAME}.search.windows.net/indexes/hr-policies/docs?api-version=2024-07-01&search=*" \
  -H "api-key: ${SEARCH_KEY}"
```

Should return 6 policy documents.

---

## 🛠️ Troubleshooting

### Deployment Failed

**View error details:**
```bash
az deployment group show \
  --resource-group rg-hr-assistant-rai \
  --name deploy-arm
```

**Common issues:**
- **Location not available**: Change `--parameters location=westus`
- **Quota exceeded**: Reduce capacity or request quota increase
- **Name already exists**: Change resourceBaseName parameter

### Search Index Not Created

**Cause:** ARM template doesn't create index structure (Azure limitation)

**Fix:** Follow Step 2 manually via Portal or Azure CLI

### Documents Not Uploaded

**Cause:** Manual step required

**Fix:** Follow Step 3 to upload sample-policies.json

---

## 💰 Cost Estimate

Same as portal deployment:
- **Azure OpenAI**: ~$0.01-0.03 per run
- **Azure AI Search**: 
  - Free: $0 (testing only)
  - Basic: ~$75/month
- **Content Safety**: Free tier (4000 calls/month)

**Total testing**: < $5  
**Total production**: ~$90/month

---

## 🗑️ Cleanup

Delete everything:

```bash
az group delete \
  --name rg-hr-assistant-rai \
  --yes \
  --no-wait
```

This removes all resources and stops all costs.

---

## 📋 ARM Template Parameters

| Parameter | Description | Default | Options |
|-----------|-------------|---------|---------|
| resourceBaseName | Base name for resources | Auto-generated | Any valid name |
| location | Azure region | Resource group location | eastus, westus, etc. |
| openAIModelName | Model to deploy | gpt-4o | gpt-4o, gpt-4o-mini |
| openAIModelCapacity | TPM capacity | 10 | 1-100 |
| searchServiceSku | Search tier | basic | free, basic, standard |
| contentSafetySku | Content Safety tier | F0 (free) | F0, S0 |

---

## ✅ Deployment Checklist

- [ ] Azure CLI or Cloud Shell ready
- [ ] Resource group created
- [ ] ARM template deployed
- [ ] All outputs captured
- [ ] API keys retrieved for all 3 services
- [ ] Search index `hr-policies` created
- [ ] 6 policy documents uploaded
- [ ] Environment variables set
- [ ] Dependencies installed (`pip install -r requirements.txt`)
- [ ] Demo runs successfully

---

## 🎯 Next Steps

1. ✅ All services deployed via ARM
2. ✅ Manual steps completed
3. ✅ Demo working
4. Compare with [hr-assistant-without-rai](https://github.com/CyberAutomatedSystems/hr-assistant-without-rai)
5. See the 4-layer RAI protection in action!

**Ready for your presentation!** 🚀
