# Chapter 3: Azure Service Operator v2

In this chapter, you'll learn how to manage Azure resources directly from Kubernetes using Azure Service Operator v2 (ASO). This enables a GitOps-friendly approach to Azure infrastructure management.

## 🎯 Learning Objectives

- Understand Kubernetes operators and the operator pattern
- Install and configure Azure Service Operator v2
- Create Azure resources using Kubernetes manifests
- Manage resource lifecycle from Kubernetes
- Connect Kubernetes workloads to Azure services

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- AKS cluster with workload identity enabled
- Azure subscription with permissions to create resources

## 🔄 Load Your Configuration

```bash
# Load your lab configuration
source ./lab-config.sh

# Verify cluster access
kubectl get nodes

# Verify OIDC issuer
echo "OIDC Issuer: $AKS_OIDC_ISSUER"
```

## 📖 Understanding Azure Service Operator

Azure Service Operator (ASO) is a Kubernetes operator that lets you manage Azure resources using Kubernetes Custom Resources (CRDs).

Benefits:
- **GitOps-friendly**: Manage Azure resources with YAML in Git
- **Kubernetes-native**: Use kubectl, Helm, Kustomize, etc.
- **Unified workflow**: Manage both K8s and Azure resources together
- **Declarative**: Desired state reconciliation

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AKS Cluster                                  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                  Azure Service Operator                       │  │
│  │                                                               │  │
│  │  kubectl apply -f storage.yaml                                │  │
│  │         │                                                     │  │
│  │         ▼                                                     │  │
│  │  ┌─────────────────┐    ┌─────────────────────────────────┐  │  │
│  │  │ StorageAccount  │───►│   ASO Controller                │  │  │
│  │  │   (K8s CR)      │    │   (Reconciliation Loop)         │  │  │
│  │  └─────────────────┘    └─────────────┬───────────────────┘  │  │
│  │                                       │                       │  │
│  └───────────────────────────────────────┼───────────────────────┘  │
│                                          │                          │
└──────────────────────────────────────────┼──────────────────────────┘
                                           │
                                           ▼
                           ┌───────────────────────────────┐
                           │         Azure                  │
                           │  ┌─────────────────────────┐  │
                           │  │   Storage Account       │  │
                           │  │   Redis Cache           │  │
                           │  │   Service Bus           │  │
                           │  │   etc.                  │  │
                           │  └─────────────────────────┘  │
                           └───────────────────────────────┘
```

> **⚠️ Important Distinction**: Azure Service Operator manages **resource lifecycle**, not policy enforcement.
> - Use **Kyverno / Gatekeeper** for controlling *who can create what* (admission control)
> - Use **ASO** for *how Azure resources are created and reconciled* (infrastructure provisioning)
>
> ASO complements policy engines — it doesn't replace them.

## 📦 Step 1: Create a User-Assigned Managed Identity

ASO needs an identity to authenticate with Azure:

```bash
# Create a managed identity for ASO
export ASO_IDENTITY_NAME="aso-identity-$STUDENT_INITIALS"

az identity create \
  --name $ASO_IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

echo "⏳ Waiting for identity to propagate..."
sleep 30

# Get identity details
export ASO_CLIENT_ID=$(az identity show \
  --name $ASO_IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --query clientId -o tsv)

export ASO_OBJECT_ID=$(az identity show \
  --name $ASO_IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --query principalId -o tsv)

echo "ASO Identity Client ID: $ASO_CLIENT_ID"
echo "ASO Identity Object ID: $ASO_OBJECT_ID"
```

## 🔐 Step 2: Assign Permissions to the Identity

```bash
# Get subscription ID
export SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# Assign Contributor role to the resource group
az role assignment create \
  --role "Contributor" \
  --assignee-object-id $ASO_OBJECT_ID \
  --assignee-principal-type ServicePrincipal \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"

echo "✅ Contributor role assigned to ASO identity"
```

## 🔗 Step 3: Create Federated Identity Credential

```bash
# Create federated credential for workload identity
az identity federated-credential create \
  --name "aso-federated-credential" \
  --identity-name $ASO_IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --issuer $AKS_OIDC_ISSUER \
  --subject "system:serviceaccount:azureserviceoperator-system:azureserviceoperator-default" \
  --audiences "api://AzureADTokenExchange"

echo "✅ Federated credential created"
```

## 📦 Step 4: Install Azure Service Operator v2

```bash
# Install cert-manager (required by ASO)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

echo "⏳ Waiting for cert-manager to be ready..."
kubectl wait --for=condition=Available deployment --all -n cert-manager --timeout=300s

# Install ASO using Helm
helm repo add aso2 https://raw.githubusercontent.com/Azure/azure-service-operator/main/v2/charts
helm repo update

# Install ASO with workload identity
helm upgrade --install aso2 aso2/azure-service-operator \
  --namespace azureserviceoperator-system \
  --create-namespace \
  --set azureSubscriptionID=$SUBSCRIPTION_ID \
  --set azureTenantID=$(az account show --query tenantId -o tsv) \
  --set azureClientID=$ASO_CLIENT_ID \
  --set useWorkloadIdentityAuth=true \
  --set crdPattern='resources.azure.com/*;storage.azure.com/*;cache.azure.com/*;servicebus.azure.com/*'

echo "⏳ Waiting for ASO to be ready..."
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=azure-service-operator -n azureserviceoperator-system --timeout=300s

echo "✅ Azure Service Operator v2 installed"
```

## 🔍 Step 5: Verify ASO Installation

```bash
# Check ASO pods
kubectl get pods -n azureserviceoperator-system

# Check installed CRDs (should see Azure resource types)
kubectl get crds | grep azure | head -20

# List available Azure resources
echo "Available Azure Resource Types:"
kubectl api-resources | grep azure | head -10
```

## 📋 Step 6: Create a Namespace for ASO Resources

```bash
# Create namespace for ASO-managed resources
kubectl create namespace aso-demo

echo "✅ Namespace 'aso-demo' created"
```

## 🏗️ Step 7: Create a ResourceGroup Reference in Kubernetes

ASO requires a Kubernetes ResourceGroup resource to reference the Azure resource group. We'll create one that references our existing resource group:

```bash
# Create a ResourceGroup resource that references the existing Azure RG
cat <<EOF | kubectl apply -f -
apiVersion: resources.azure.com/v1api20200601
kind: ResourceGroup
metadata:
  name: $RESOURCE_GROUP
  namespace: aso-demo
spec:
  location: $LOCATION
  azureName: $RESOURCE_GROUP
EOF

echo "⏳ Waiting for ResourceGroup to be ready..."
kubectl wait --for=condition=Ready resourcegroup/$RESOURCE_GROUP -n aso-demo --timeout=120s

echo "✅ ResourceGroup reference created"
```

> **Note**: This creates a Kubernetes resource that represents your existing Azure resource group. ASO will recognize the existing resource group and manage it going forward.

## 🗄️ Step 8: Create an Azure Storage Account

```bash
# Create a unique storage account name
export STORAGE_ACCOUNT_NAME="staso${STUDENT_INITIALS}$(date +%s | tail -c 5)"

# Create Storage Account using ASO
cat <<EOF | kubectl apply -f -
apiVersion: storage.azure.com/v1api20230101
kind: StorageAccount
metadata:
  name: $STORAGE_ACCOUNT_NAME
  namespace: aso-demo
spec:
  location: $LOCATION
  owner:
    name: $RESOURCE_GROUP
  sku:
    name: Standard_LRS
  kind: StorageV2
  accessTier: Hot
EOF

echo "⏳ Storage Account creation initiated..."
echo "Storage Account Name: $STORAGE_ACCOUNT_NAME"
```

## 🔍 Step 9: Monitor Resource Creation

```bash
# Watch the resource status
kubectl get storageaccount -n aso-demo -w &
WATCH_PID=$!

# Wait for the resource to be ready
echo "⏳ Waiting for Storage Account to be ready..."
kubectl wait --for=condition=Ready storageaccount/$STORAGE_ACCOUNT_NAME -n aso-demo --timeout=300s

# Stop the watch
kill $WATCH_PID 2>/dev/null

# Check the resource status
kubectl get storageaccount -n aso-demo

# Describe the resource for details
kubectl describe storageaccount $STORAGE_ACCOUNT_NAME -n aso-demo
```

## ✅ Step 10: Verify in Azure Portal

```bash
# Verify the storage account exists in Azure
az storage account show \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "{Name:name, Location:location, Sku:sku.name, Status:provisioningState}" \
  -o table

echo "✅ Storage Account verified in Azure"
```

## 📦 Step 11: Create a Blob Service and Container

ASO follows the Azure resource hierarchy: **StorageAccount → BlobService → Container**. We need to create the BlobService first:

```bash
# First, create the Blob Service (required parent for containers)
cat <<EOF | kubectl apply -f -
apiVersion: storage.azure.com/v1api20230101
kind: StorageAccountsBlobService
metadata:
  name: ${STORAGE_ACCOUNT_NAME}-blobservice
  namespace: aso-demo
spec:
  owner:
    name: $STORAGE_ACCOUNT_NAME
EOF

echo "⏳ Waiting for Blob Service to be ready..."
kubectl wait --for=condition=Ready storageaccountsblobservice/${STORAGE_ACCOUNT_NAME}-blobservice -n aso-demo --timeout=120s

echo "✅ Blob Service created"

# Now create the blob container
cat <<EOF | kubectl apply -f -
apiVersion: storage.azure.com/v1api20230101
kind: StorageAccountsBlobServicesContainer
metadata:
  name: my-container
  namespace: aso-demo
spec:
  owner:
    name: ${STORAGE_ACCOUNT_NAME}-blobservice
  publicAccess: None
EOF

echo "⏳ Blob container creation initiated..."

# Wait for it to be ready
kubectl wait --for=condition=Ready storageaccountsblobservicescontainer/my-container -n aso-demo --timeout=120s

echo "✅ Blob container created"
```

> **Note**: The resource hierarchy in ASO mirrors Azure's ARM hierarchy. A container's owner must be a BlobService, not the StorageAccount directly.

## 🔑 Step 12: Export Secrets to Kubernetes

ASO can export storage account keys to Kubernetes secrets. First, let's check what secret fields are available:

```bash
# View the StorageAccount CRD to see available secret fields
kubectl get crd storageaccounts.storage.azure.com -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.operatorSpec.properties.secrets.properties}' | jq 'keys'
```

Now export the storage account keys:

```bash
# Update the storage account to export secrets (key1 and key2 are the valid fields)
cat <<EOF | kubectl apply -f -
apiVersion: storage.azure.com/v1api20230101
kind: StorageAccount
metadata:
  name: $STORAGE_ACCOUNT_NAME
  namespace: aso-demo
spec:
  location: $LOCATION
  owner:
    name: $RESOURCE_GROUP
  sku:
    name: Standard_LRS
  kind: StorageV2
  accessTier: Hot
  operatorSpec:
    secrets:
      key1:
        name: storage-account-secret
        key: storageKey
EOF

echo "⏳ Waiting for secret to be created..."
sleep 30

# Check if the secret was created
kubectl get secret storage-account-secret -n aso-demo

# View the secret keys (not values)
kubectl get secret storage-account-secret -n aso-demo -o jsonpath='{.data}' | jq 'keys'

echo "✅ Storage account key exported to Kubernetes"
```

> **Note**: ASO exports storage account keys, not connection strings. You'll need to construct the connection string in your application or use the storage account name + key directly.

## 🚀 Step 13: Use the Secret in a Pod

```bash
# Create a pod that uses the storage account secret
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: storage-test-pod
  namespace: aso-demo
spec:
  containers:
  - name: azure-cli
    image: mcr.microsoft.com/azure-cli:latest
    command: ["sleep", "3600"]
    env:
    - name: AZURE_STORAGE_ACCOUNT
      value: "$STORAGE_ACCOUNT_NAME"
    - name: AZURE_STORAGE_KEY
      valueFrom:
        secretKeyRef:
          name: storage-account-secret
          key: storageKey
EOF

echo "⏳ Waiting for pod to be ready..."
kubectl wait --for=condition=Ready pod/storage-test-pod -n aso-demo --timeout=120s

echo "✅ Test pod created with storage account credentials"
```

## 🧪 Step 14: Test Storage Access from Pod

```bash
# Exec into the pod and test storage access using the environment variables
kubectl exec -it -n aso-demo storage-test-pod -- /bin/bash -c 'az storage container list --account-name $AZURE_STORAGE_ACCOUNT --account-key $AZURE_STORAGE_KEY --output table'

echo "✅ Storage access verified from within the pod"
```

If the above doesn't work, you can also test interactively:

```bash
# Get into the pod
kubectl exec -it -n aso-demo storage-test-pod -- /bin/bash

# Inside the pod, verify environment variables are set
echo "Account: $AZURE_STORAGE_ACCOUNT"
echo "Key is set: $([ -n "$AZURE_STORAGE_KEY" ] && echo 'yes' || echo 'no')"

# List containers using environment variables
az storage container list --account-name $AZURE_STORAGE_ACCOUNT --account-key $AZURE_STORAGE_KEY --output table

# Exit the pod
exit
```

## 📦 Step 15: Create a Redis Cache (Optional - Takes ~20 mins)

```bash
# Create Azure Redis Cache
export REDIS_NAME="redis-$STUDENT_INITIALS-$(date +%s | tail -c 5)"

cat <<EOF | kubectl apply -f -
apiVersion: cache.azure.com/v1api20230401
kind: Redis
metadata:
  name: $REDIS_NAME
  namespace: aso-demo
spec:
  location: $LOCATION
  owner:
    name: $RESOURCE_GROUP
  sku:
    name: Basic
    family: C
    capacity: 0
  enableNonSslPort: false
  minimumTlsVersion: "1.2"
  operatorSpec:
    secrets:
      primaryKey:
        name: redis-secret
        key: primaryKey
      hostName:
        name: redis-secret
        key: hostName
EOF

echo "⏳ Redis Cache creation initiated (this takes ~15-20 minutes)..."
echo "Redis Name: $REDIS_NAME"
echo ""
echo "You can continue to the next step while Redis provisions."
echo "Monitor with: kubectl get redis -n aso-demo -w"
```

## 📊 Step 16: View All ASO Resources

```bash
# List all ASO-managed resources
echo "========================================"
echo "  ASO-MANAGED RESOURCES"
echo "========================================"

echo ""
echo "Storage Accounts:"
kubectl get storageaccounts -n aso-demo

echo ""
echo "Blob Containers:"
kubectl get storageaccountsblobservicescontainers -n aso-demo

echo ""
echo "Redis Caches:"
kubectl get redis -n aso-demo

echo ""
echo "All ASO Resources:"
kubectl get all -n aso-demo
```

## �️ Step 17: Delete a Resource

When you delete the Kubernetes resource, ASO deletes the Azure resource:

```bash
# Delete the blob container
kubectl delete storageaccountsblobservicescontainer my-container -n aso-demo

echo "⏳ Waiting for container deletion..."
sleep 30

# Verify it's deleted in Azure
az storage container list \
  --account-name $STORAGE_ACCOUNT_NAME \
  --auth-mode login 2>/dev/null || echo "Container deleted or no containers found"

echo "✅ Blob container deleted"
```

## 📝 Step 18: Save ASO Configuration

```bash
# Update lab config with ASO details
cat >> ./lab-config.sh <<EOF

# ASO Configuration
export ASO_IDENTITY_NAME="$ASO_IDENTITY_NAME"
export ASO_CLIENT_ID="$ASO_CLIENT_ID"
export STORAGE_ACCOUNT_NAME="$STORAGE_ACCOUNT_NAME"
export REDIS_NAME="$REDIS_NAME"
EOF

echo "✅ ASO configuration saved"
```

## 🎓 Summary

You have successfully:
- ✅ Installed Azure Service Operator v2 with Workload Identity
- ✅ Created Azure resources using Kubernetes manifests
- ✅ Exported secrets to Kubernetes for application use
- ✅ Connected workloads to Azure services
- ✅ Managed the full resource lifecycle from Kubernetes

## 📝 Important Notes

- **Deletion Policy**: By default, deleting K8s resources deletes Azure resources. Use `operatorSpec.reconcilePolicy` to control this behavior — set to `skip` to prevent accidental Azure resource deletion during `kubectl delete`.
- **Adoption**: ASO can adopt existing Azure resources
- **CRD Patterns**: Install only the CRDs you need for faster startup
- **Permissions**: Use least-privilege RBAC for production
- **Security Best Practice**: In production, prefer workload identity-based SDK auth (as used in this lab) over exporting storage keys to secrets. Key-based auth is shown here for learning purposes.

## 🔍 Troubleshooting

```bash
# Check ASO controller logs
kubectl logs -n azureserviceoperator-system -l app.kubernetes.io/name=azure-service-operator -f

# Check resource conditions
kubectl describe storageaccount $STORAGE_ACCOUNT_NAME -n aso-demo

# Check for failed resources
kubectl get storageaccount -n aso-demo -o jsonpath='{range .items[*]}{.metadata.name}: {.status.conditions[*].message}{"\n"}{end}'

# Verify identity permissions
az role assignment list \
  --assignee $ASO_OBJECT_ID \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP" \
  -o table
```

## 🚀 Next Steps

Continue to **[Chapter 4: Crossplane](../chapter-4-crossplane/README.md)** to learn about a multi-cloud alternative for managing cloud resources from Kubernetes.

---

**Questions or Issues?**
- Check ASO controller logs for errors
- Verify managed identity has correct permissions
- Ensure federated credential is properly configured
- Ask your instructor for assistance

---

## ⚠️ Disclaimer

Educational/lab purposes only. Calculations and/or statements may contain errors.
