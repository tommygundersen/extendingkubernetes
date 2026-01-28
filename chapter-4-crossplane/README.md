# Chapter 4: Crossplane

In this chapter, you'll learn how to use Crossplane to manage cloud resources from Kubernetes. Crossplane provides a universal control plane that works across multiple cloud providers and enables powerful composition patterns.

## 🎯 Learning Objectives

- Understand Crossplane architecture and concepts
- Install Crossplane and the Azure Provider
- Create Managed Resources directly
- Build Composite Resources using XRDs and Compositions
- Compare Crossplane with Azure Service Operator

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- Completed [Chapter 3: Azure Service Operator v2](../chapter-3-aso/README.md)
- AKS cluster running with workload identity

## 🔄 Load Your Configuration

```bash
# Load your lab configuration
source ./lab-config.sh

# Verify cluster access
kubectl get nodes
```

## 📖 Understanding Crossplane

Crossplane extends Kubernetes to become a universal control plane for cloud infrastructure. Key concepts:

- **Provider**: Plugin that knows how to manage a specific cloud (Azure, AWS, GCP)
- **Managed Resource (MR)**: Direct representation of a cloud resource
- **Composite Resource Definition (XRD)**: Schema for custom abstractions
- **Composition**: How to build infrastructure from multiple resources
- **Composite Resource (XR)**: Instance of your custom abstraction
- **Claim**: Namespace-scoped way to request infrastructure

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              Crossplane                                     │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                        Platform Team                                 │  │
│  │                                                                      │  │
│  │  ┌────────────────────┐    ┌────────────────────────────────────┐   │  │
│  │  │  Composite Resource │   │         Composition                │   │  │
│  │  │  Definition (XRD)   │──►│  (How to build the infrastructure) │   │  │
│  │  │  (The schema/API)   │   │                                    │   │  │
│  │  └────────────────────┘   └───────────────┬────────────────────┘   │  │
│  │                                           │                         │  │
│  └───────────────────────────────────────────┼─────────────────────────┘  │
│                                              │                            │
│  ┌───────────────────────────────────────────┼─────────────────────────┐  │
│  │                        Application Team   │                          │  │
│  │                                           ▼                          │  │
│  │  ┌────────────────────┐    ┌─────────────────────────────────────┐  │  │
│  │  │       Claim        │───►│      Composite Resource (XR)        │  │  │
│  │  │  (I need a DB!)    │    │   (Actual infrastructure bundle)    │  │  │
│  │  └────────────────────┘    └───────────────┬─────────────────────┘  │  │
│  │                                            │                         │  │
│  └────────────────────────────────────────────┼─────────────────────────┘  │
│                                               │                            │
│                                               ▼                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                     Managed Resources                                │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │  │
│  │  │    Storage   │  │    Redis     │  │  Networking  │  ...          │  │
│  │  │    Account   │  │    Cache     │  │    Rules     │               │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘               │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                               │                            │
└───────────────────────────────────────────────┼────────────────────────────┘
                                                │
                                                ▼
                               ┌────────────────────────────────┐
                               │         Azure / AWS / GCP      │
                               └────────────────────────────────┘
```

> **⚠️ Important Distinction**: Crossplane is **not** a policy engine and **not** a deployment tool.
> It manages **resource lifecycle** and **platform abstractions**.
>
> | Tool | Purpose |
> |------|--------|
> | Kyverno / Gatekeeper | Policy & guardrails |
> | ASO | Azure-native resource lifecycle |
> | Crossplane | Platform abstractions & multi-cloud control plane |

> **🧠 Reality Check**: Crossplane shines when you need **standardized abstractions across many teams or clouds**.
> If you only need direct Azure resources in AKS, ASO is often simpler and cheaper to operate.
> Choose based on your actual multi-cloud/platform-team needs, not theoretical future requirements.

## 📦 Step 1: Install Crossplane

```bash
# Create namespace for Crossplane
kubectl create namespace crossplane-system

# Install Crossplane using Helm
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --set args='{"--enable-usages"}'

echo "⏳ Waiting for Crossplane to be ready..."
kubectl wait --for=condition=ready pod -l app=crossplane -n crossplane-system --timeout=300s

echo "✅ Crossplane installed successfully"
```

## 🔍 Step 2: Verify Crossplane Installation

```bash
# Check Crossplane pods
kubectl get pods -n crossplane-system

# Check Crossplane CRDs
kubectl get crds | grep crossplane

# Verify Crossplane is ready
kubectl get deployment -n crossplane-system
```

## 🔌 Step 3: Create a Service Principal for Crossplane

For this lab, we'll use a Service Principal with client secret for simpler authentication:

```bash
# Get tenant and subscription IDs
export TENANT_ID=$(az account show --query tenantId -o tsv)
export SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# Create a Service Principal for Crossplane
export CROSSPLANE_SP_NAME="sp-crossplane-$STUDENT_INITIALS"

az ad sp create-for-rbac \
  --name $CROSSPLANE_SP_NAME \
  --role Contributor \
  --scopes "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP" \
  --output json > crossplane-sp.json

# Extract credentials
export CROSSPLANE_CLIENT_ID=$(jq -r '.appId' crossplane-sp.json)
export CROSSPLANE_CLIENT_SECRET=$(jq -r '.password' crossplane-sp.json)

echo "Crossplane Service Principal Client ID: $CROSSPLANE_CLIENT_ID"
echo "✅ Service Principal created"
```

> **Note**: In production, you should use Workload Identity instead of client secrets. For this lab, we use a Service Principal for simplicity.

> **🎯 Optional Exercise**: After completing this chapter, try re-configuring the Azure provider to use Workload Identity (similar to Chapter 3's ASO setup) for architectural consistency across your operators.

## 📦 Step 4: Install Azure Provider

```bash
# Install the Azure Provider for Crossplane
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure-storage
spec:
  package: xpkg.upbound.io/upbound/provider-azure-storage:v1.1.0
EOF

echo "⏳ Waiting for provider to be installed..."
# Use the full resource name to avoid conflict with Gatekeeper's provider CRD
kubectl wait --for=condition=healthy provider.pkg.crossplane.io/provider-azure-storage --timeout=300s

echo "✅ Azure Storage Provider installed"
```

> **Note**: We use `provider.pkg.crossplane.io` to specify the Crossplane provider CRD explicitly, since Gatekeeper also has a `providers` CRD.

## 🔍 Step 5: Verify Provider Installation

```bash
# Check provider status (use full resource name to avoid Gatekeeper conflict)
kubectl get provider.pkg.crossplane.io

# Check provider pods
kubectl get pods -n crossplane-system | grep provider

# Check available CRDs from the provider
kubectl get crds | grep azure.upbound.io | head -10
```

Expected output for providers:
```
NAME                       INSTALLED   HEALTHY   PACKAGE                                                  AGE
provider-azure-storage     True        True      xpkg.upbound.io/upbound/provider-azure-storage:v1.1.0   Xm
```

## 🔐 Step 6: Create Provider Configuration with Secret

```bash
# Create a Kubernetes secret with Azure credentials
kubectl create secret generic azure-creds \
  -n crossplane-system \
  --from-literal=credentials="{
  \"clientId\": \"$CROSSPLANE_CLIENT_ID\",
  \"clientSecret\": \"$CROSSPLANE_CLIENT_SECRET\",
  \"subscriptionId\": \"$SUBSCRIPTION_ID\",
  \"tenantId\": \"$TENANT_ID\"
}"

echo "✅ Azure credentials secret created"

# Create the ProviderConfig for Azure
cat <<EOF | kubectl apply -f -
apiVersion: azure.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: azure-creds
      key: credentials
EOF

echo "✅ ProviderConfig created"
```

## 📋 Step 7: Create a Namespace for Crossplane Resources

```bash
# Create namespace for Crossplane-managed resources
kubectl create namespace crossplane-demo

echo "✅ Namespace 'crossplane-demo' created"
```

## 🗄️ Step 8: Create a Managed Resource (Storage Account)

```bash
# Create a unique storage account name
export XP_STORAGE_NAME="stxp${STUDENT_INITIALS}$(date +%s | tail -c 5)"

# Create a Storage Account using Crossplane
cat <<EOF | kubectl apply -f -
apiVersion: storage.azure.upbound.io/v1beta1
kind: Account
metadata:
  name: $XP_STORAGE_NAME
spec:
  forProvider:
    accountReplicationType: LRS
    accountTier: Standard
    location: $LOCATION
    resourceGroupName: $RESOURCE_GROUP
  providerConfigRef:
    name: default
EOF

echo "⏳ Storage Account creation initiated..."
echo "Storage Account Name: $XP_STORAGE_NAME"
```

## 🔍 Step 9: Monitor Resource Creation

```bash
# Watch the resource status
echo "⏳ Waiting for Storage Account to be ready..."
kubectl wait --for=condition=Ready account/$XP_STORAGE_NAME --timeout=300s

# Check the resource status
kubectl get account

# Get detailed information
kubectl describe account $XP_STORAGE_NAME
```

## ✅ Step 10: Verify in Azure

```bash
# Verify the storage account exists in Azure
az storage account show \
  --name $XP_STORAGE_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "{Name:name, Location:location, Status:provisioningState}" \
  -o table

echo "✅ Storage Account verified in Azure"
```

## 📐 Step 11: Create a Composite Resource Definition (XRD)

Now let's create an abstraction that platform teams can offer to developers:

```bash
# First, install the patch-and-transform function (required for Compositions in newer Crossplane)
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-patch-and-transform:v0.4.0
EOF

echo "⏳ Waiting for function to be ready..."
kubectl wait --for=condition=healthy function/function-patch-and-transform --timeout=120s

echo "✅ Patch-and-transform function installed"

# Create an XRD for a "StorageBundle" (using v1 to support Claims)
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xstoragebundles.demo.crossplane.io
spec:
  group: demo.crossplane.io
  names:
    kind: XStorageBundle
    plural: xstoragebundles
  claimNames:
    kind: StorageBundle
    plural: storagebundles
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              parameters:
                type: object
                properties:
                  location:
                    type: string
                    description: "Azure region for resources"
                required:
                - location
            required:
            - parameters
EOF

echo "✅ XRD 'XStorageBundle' created"
```

## 🏗️ Step 12: Create a Composition

The Composition defines what resources to create when someone requests a StorageBundle. In newer Crossplane versions, Compositions use a pipeline mode with functions:

```bash
# Create a Composition for the StorageBundle using pipeline mode
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: storagebundle-azure-standard
spec:
  compositeTypeRef:
    apiVersion: demo.crossplane.io/v1alpha1
    kind: XStorageBundle
  mode: Pipeline
  pipeline:
  - step: patch-and-transform
    functionRef:
      name: function-patch-and-transform
    input:
      apiVersion: pt.fn.crossplane.io/v1beta1
      kind: Resources
      resources:
      - name: storage-account
        base:
          apiVersion: storage.azure.upbound.io/v1beta1
          kind: Account
          metadata:
            name: stxp$STUDENT_INITIALS
          spec:
            forProvider:
              accountReplicationType: LRS
              accountTier: Standard
              resourceGroupName: $RESOURCE_GROUP
            providerConfigRef:
              name: default
        patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.location
          toFieldPath: spec.forProvider.location
EOF

echo "✅ Composition 'storagebundle-azure-standard' created"
```

> **Note**: Crossplane uses Pipeline mode with Functions for Compositions. Storage account names must be 3-24 characters, lowercase alphanumeric only.

## 🎯 Step 13: Create a Claim (Developer Experience)

Now a developer can request infrastructure using a simple Claim:

```bash
# Create a StorageBundle claim
cat <<EOF | kubectl apply -f -
apiVersion: demo.crossplane.io/v1alpha1
kind: StorageBundle
metadata:
  name: my-app-storage
  namespace: crossplane-demo
spec:
  parameters:
    location: $LOCATION
EOF

echo "⏳ StorageBundle claim submitted..."
```

## 🔍 Step 14: Monitor the Composite Resource

```bash
# Check the claim
kubectl get storagebundle -n crossplane-demo

# Check the composite resource (XR)
kubectl get xstoragebundle

# Check the managed resources created
kubectl get account | grep st

# Get detailed status
kubectl describe storagebundle my-app-storage -n crossplane-demo

echo "⏳ Waiting for all resources to be ready..."
sleep 60

# Check status again
kubectl get storagebundle -n crossplane-demo
kubectl get account
```

## 📊 Step 15: View the Resource Hierarchy

```bash
echo "========================================"
echo "  CROSSPLANE RESOURCE HIERARCHY"
echo "========================================"

echo ""
echo "1. Claim (namespace-scoped, for developers):"
kubectl get storagebundle -n crossplane-demo

echo ""
echo "2. Composite Resource (cluster-scoped):"
kubectl get xstoragebundle

echo ""
echo "3. Managed Resources (actual cloud resources):"
kubectl get account

echo ""
echo "4. Azure Resources:"
az storage account list \
  --resource-group $RESOURCE_GROUP \
  --query "[?starts_with(name, 'st')].{Name:name, Location:location, Status:provisioningState}" \
  -o table
```

## 📋 Step 16: Compare ASO vs Crossplane

```bash
echo "========================================"
echo "  ASO vs CROSSPLANE COMPARISON"
echo "========================================"
echo ""
echo "Azure Service Operator (ASO):"
echo "  ✓ First-party Microsoft support"
echo "  ✓ Deep Azure integration"
echo "  ✓ Automatic secret export"
echo "  ✓ Azure-specific features"
kubectl get storageaccounts -n aso-demo 2>/dev/null || echo "  (No ASO resources in aso-demo)"

echo ""
echo "Crossplane:"
echo "  ✓ Multi-cloud support"
echo "  ✓ Composition abstractions"
echo "  ✓ Platform team tooling"
echo "  ✓ Provider ecosystem"
kubectl get account 2>/dev/null || echo "  (No Crossplane accounts found)"
```

| Feature        | Azure Service Operator | Crossplane           |
| -------------- | ---------------------- | -------------------- |
| Multi-Cloud    | Azure only             | AWS, Azure, GCP, +   |
| Abstraction    | 1:1 Azure resources    | Compositions & XRDs  |
| Secret Export  | Built-in               | External Secrets     |
| Learning Curve | Medium                 | High                 |
| Use Case       | Azure-focused teams    | Platform engineering |

> **🔄 Coexistence**: ASO and Crossplane **can run side-by-side** in the same cluster. Some teams use ASO for simple Azure resources and Crossplane for multi-cloud abstractions. Choose the right tool for each job.

## 🗑️ Step 17: Cleanup Crossplane Resources

```bash
# Delete the claim (this will delete all composed resources)
kubectl delete storagebundle my-app-storage -n crossplane-demo

echo "⏳ Waiting for resources to be deleted..."
sleep 60

# Verify resources are deleted
kubectl get account
kubectl get xstoragebundle
```

## 📝 Step 18: Save Crossplane Configuration

```bash
# Update lab config with Crossplane details
cat >> ./lab-config.sh <<EOF

# Crossplane Configuration
export CROSSPLANE_SP_NAME="$CROSSPLANE_SP_NAME"
export CROSSPLANE_CLIENT_ID="$CROSSPLANE_CLIENT_ID"
export XP_STORAGE_NAME="$XP_STORAGE_NAME"
EOF

# Clean up the credentials file (contains secrets)
rm -f crossplane-sp.json

echo "✅ Crossplane configuration saved"
```

## 🎓 Summary

You have successfully:
- ✅ Installed Crossplane with the Azure Provider
- ✅ Created Managed Resources directly
- ✅ Built a Composite Resource Definition (XRD)
- ✅ Created a Composition to bundle resources
- ✅ Used Claims for developer-friendly infrastructure requests
- ✅ Compared Crossplane with Azure Service Operator

## 📝 Important Notes

- **Compositions**: Enable platform teams to hide complexity
- **Claims**: Provide namespace-scoped developer experience
- **Multiple Compositions**: One XRD can have multiple Compositions (e.g., dev vs prod, azure vs aws)
- **Patch & Transform**: Powerful system to customize resources

**⚠️ Things That Surprise People:**

| Gotcha                                             | Explanation                                                          |
| -------------------------------------------------- | -------------------------------------------------------------------- |
| Managed Resources are **cluster-scoped**           | Unlike ASO, MRs don't live in namespaces (Claims do)                 |
| Names often must be **globally unique**            | Azure storage accounts, etc. need unique names across Azure          |
| Drift reconciliation is **continuous**             | Crossplane reverts manual Azure portal changes back to desired state |
| Composition changes don't auto-update existing XRs | You may need to recreate or manually trigger updates                 |

## 🔍 Troubleshooting

```bash
# Check Crossplane controller logs
kubectl logs -n crossplane-system -l app=crossplane -f

# Check provider logs
kubectl logs -n crossplane-system -l pkg.crossplane.io/provider=provider-azure-storage -f

# Debug XR status
kubectl describe xstoragebundle

# Check managed resource conditions
kubectl describe account $XP_STORAGE_NAME

# Verify provider health
kubectl get providers
```

## 🚀 Next Steps

Continue to **[Chapter 5: Dapr Introduction](../chapter-5-dapr/README.md)** to learn about building distributed applications with Dapr.

---

**Questions or Issues?**
- Check Crossplane and provider logs
- Verify ProviderConfig credentials
- Ensure XRD and Composition reference each other correctly
- Ask your instructor for assistance

---

## ⚠️ Disclaimer

Educational/lab purposes only. Calculations and/or statements may contain errors.

This documentation is provided "as is" without warranty of any kind. The author takes no responsibility for any errors, omissions, or inaccuracies contained herein. This material may contain incorrect or outdated information. Always verify configurations against official Microsoft Azure and Kubernetes documentation before use in production environments. Use at your own risk.
