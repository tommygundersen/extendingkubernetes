# Chapter 0: Setup and Prerequisites

In this chapter, you'll set up your environment, create the resource group, and deploy the AKS cluster that will be used throughout all subsequent chapters.

## 📋 Prerequisites

Before starting this lab, ensure you have:

### Required Tools

1. **Azure CLI** (version 2.50.0 or later)
   - [Installation guide](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
   - Verify: `az --version`

2. **kubectl** (Kubernetes command-line tool)
   - [Installation guide](https://kubernetes.io/docs/tasks/tools/)
   - Verify: `kubectl version --client`

3. **Helm** (version 3.x)
   - [Installation guide](https://helm.sh/docs/intro/install/)
   - Verify: `helm version`

4. **jq** (JSON processor - optional but helpful)
   - [Installation guide](https://stedolan.github.io/jq/download/)
   - Verify: `jq --version`

### Azure Requirements

- Active Azure subscription
- Permissions to create:
  - Resource groups
  - AKS clusters
  - Azure resources (Storage, Redis, etc.)
  - Service Principals / Managed Identities

## 🎯 Learning Objectives

- Set up student-specific naming conventions
- Create a resource group for lab resources
- Deploy an AKS cluster
- Verify cluster access and functionality

## 📝 Step 1: Set Your Student Initials

To avoid naming conflicts in a shared subscription, you'll use your initials for resource naming.

```bash
# Set your initials (use 2-4 lowercase letters)
export STUDENT_INITIALS="abc"  # CHANGE THIS to your initials

# Verify it's set
echo "Your initials: $STUDENT_INITIALS"
```

## 📦 Step 2: Define Resource Names

```bash
# Set variables for resource names
export LOCATION="swedencentral"
export RESOURCE_GROUP="rg-extending-k8s-$STUDENT_INITIALS"
export CLUSTER_NAME="aks-extend-$STUDENT_INITIALS"

# Display your configuration
echo "Resource Configuration:"
echo "  Location: $LOCATION"
echo "  Resource Group: $RESOURCE_GROUP"
echo "  Cluster Name: $CLUSTER_NAME"
```

## 🔐 Step 3: Login to Azure

```bash
# Login to Azure
az login

# Set the subscription (if you have multiple)
# az account set --subscription "Your Subscription Name or ID"

# Verify subscription
az account show --query "{Name:name, ID:id}" -o table
```

## 🏗️ Step 4: Create Resource Group

```bash
# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION

echo "✅ Resource group created: $RESOURCE_GROUP"
```

## ☸️ Step 5: Create AKS Cluster

```bash
# Create AKS cluster with workload identity enabled
az aks create \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --node-count 2 \
  --node-vm-size Standard_DS2_v2 \
  --network-plugin azure \
  --enable-managed-identity \
  --enable-workload-identity \
  --enable-oidc-issuer \
  --generate-ssh-keys

echo "✅ AKS cluster created: $CLUSTER_NAME"
```

> ⏳ **Note**: This command takes approximately 5-10 minutes to complete.

## 🔌 Step 6: Configure kubectl Access

```bash
# Get AKS credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --overwrite-existing

echo "✅ kubectl configured for cluster: $CLUSTER_NAME"
```

## ✅ Step 7: Verify Cluster

```bash
# Check nodes are ready
kubectl get nodes

# Verify all system pods are running
kubectl get pods -n kube-system

# Check cluster info
kubectl cluster-info
```

Expected output for nodes:
```
NAME                                STATUS   ROLES    AGE   VERSION
aks-nodepool1-XXXXX-vmss000000     Ready    <none>   Xm    v1.XX.X
aks-nodepool1-XXXXX-vmss000001     Ready    <none>   Xm    v1.XX.X
```

## 📦 Step 8: Install Helm Repositories

We'll add the Helm repositories needed for the lab:

```bash
# Add Kyverno Helm repo
helm repo add kyverno https://kyverno.github.io/kyverno/

# Add Gatekeeper Helm repo
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts

# Add Crossplane Helm repo
helm repo add crossplane-stable https://charts.crossplane.io/stable

# Add Dapr Helm repo
helm repo add dapr https://dapr.github.io/helm-charts/

# Update all repos
helm repo update

echo "✅ Helm repositories added and updated"
```

## 📊 Step 9: Get Cluster OIDC Issuer URL

This will be needed for Azure Service Operator and other identity-based integrations:

```bash
# Get OIDC Issuer URL
export AKS_OIDC_ISSUER=$(az aks show \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)

echo "OIDC Issuer URL: $AKS_OIDC_ISSUER"
```

## 📊 Step 10: Save Your Configuration

Create a file to save your environment variables for future sessions:

```bash
# Create a configuration file
cat > ./lab-config.sh <<EOF
# Extending Kubernetes Lab Configuration for student: $STUDENT_INITIALS
export STUDENT_INITIALS="$STUDENT_INITIALS"
export LOCATION="$LOCATION"
export RESOURCE_GROUP="$RESOURCE_GROUP"
export CLUSTER_NAME="$CLUSTER_NAME"

# Derived values
export SUBSCRIPTION_ID="\$(az account show --query id -o tsv)"
export AKS_OIDC_ISSUER="\$(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query 'oidcIssuerProfile.issuerUrl' -o tsv)"

echo "Configuration loaded for student: \$STUDENT_INITIALS"
EOF

chmod +x ./lab-config.sh

echo "✅ Configuration saved to lab-config.sh"
echo "   Load it in future sessions with: source ./lab-config.sh"
```

## 🎓 Summary

You have successfully:
- ✅ Set up student-specific naming
- ✅ Created a resource group
- ✅ Deployed an AKS cluster with workload identity enabled
- ✅ Configured kubectl access
- ✅ Added Helm repositories
- ✅ Saved your configuration

## 📝 Important Notes

- **Keep your cluster running** - You'll use it for all subsequent chapters
- **Save your lab-config.sh file** - You'll need it to restore variables
- **Note your resource names** - You'll reference them throughout the lab

## 🔄 Loading Configuration in Future Sessions

When you start a new terminal session:

```bash
# Navigate to lab directory
cd /path/to/extendingkubernetes

# Load your configuration
source ./lab-config.sh

# Re-authenticate if needed
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME
```

## 🧹 Cleanup (Only at End of ALL Chapters)

**⚠️ WARNING: Only run this after completing ALL chapters!**

```bash
# Delete the entire resource group (removes all resources)
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait

echo "🗑️ Resource group deletion initiated"
```

## 🚀 Next Steps

Now that your environment is set up, proceed to:
- **[Chapter 1: Policy Engines - Kyverno](../chapter-1-kyverno/README.md)**

---

**Questions or Issues?**
- Verify all commands completed successfully
- Check that all pods in kube-system are running
- Ensure you have proper Azure permissions
- Ask your instructor for assistance

---

## ⚠️ Disclaimer

Educational/lab purposes only. Calculations and/or statements may contain errors.

This documentation is provided "as is" without warranty of any kind. The author takes no responsibility for any errors, omissions, or inaccuracies contained herein. This material may contain incorrect or outdated information. Always verify configurations against official Microsoft Azure and Kubernetes documentation before use in production environments. Use at your own risk.
