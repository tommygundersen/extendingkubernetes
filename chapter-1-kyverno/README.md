# Chapter 1: Policy Engines - Kyverno

In this chapter, you'll learn how to enforce policies in Kubernetes using Kyverno, a Kubernetes-native policy engine that uses YAML for policy definitions.

## 🎯 Learning Objectives

- Understand Kubernetes admission controllers
- Install and configure Kyverno
- Create validation policies to enforce standards
- Create mutation policies to modify resources automatically
- Test policy enforcement with real scenarios

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- AKS cluster running
- kubectl configured

## 🔄 Load Your Configuration

```bash
# Load your lab configuration
source ./lab-config.sh

# Verify cluster access
kubectl get nodes
```

## 📖 Understanding Kyverno

Kyverno is a policy engine designed specifically for Kubernetes. It allows you to:

- **Validate** resources against policies (allow/deny)
- **Mutate** resources by automatically modifying them
- **Generate** new resources based on triggers
- **Verify Images** for supply chain security

Unlike OPA/Gatekeeper, Kyverno uses native Kubernetes YAML syntax, making it easier to learn and adopt.

```
                    ┌────────────────────────────────────────┐
                    │           Kubernetes API Server        │
                    │                                        │
   kubectl apply   │  ┌──────────────────────────────────┐  │
   ──────────────► │  │      Admission Webhook           │  │
                   │  │                                   │  │
                   │  │  ┌─────────────────────────────┐ │  │
                   │  │  │         Kyverno             │ │  │
                   │  │  │  ┌─────────┐ ┌───────────┐  │ │  │
                   │  │  │  │Validate │ │  Mutate   │  │ │  │
                   │  │  │  └─────────┘ └───────────┘  │ │  │
                   │  │  └─────────────────────────────┘ │  │
                   │  └──────────────────────────────────┘  │
                   │                    │                   │
                   │                    ▼                   │
                   │  ┌──────────────────────────────────┐  │
                   │  │         etcd (storage)           │  │
                   │  └──────────────────────────────────┘  │
                   └────────────────────────────────────────┘
```

> **🔑 Key Concept**: Kyverno runs as a Kubernetes admission controller using **validating and mutating admission webhooks**. This means:
> - Policies are evaluated **synchronously** during `kubectl apply`
> - `Enforce` mode policies **block** resource creation before it reaches etcd
> - `Mutate` policies **modify** objects before they are persisted
> - Errors appear immediately at the command line when policies are violated

## 📦 Step 1: Install Kyverno

```bash
# Create namespace for Kyverno
kubectl create namespace kyverno

# Install Kyverno using Helm
helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --set replicaCount=1

echo "⏳ Waiting for Kyverno to be ready..."
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=kyverno -n kyverno --timeout=300s

echo "✅ Kyverno installed successfully"
```

## 🔍 Step 2: Verify Kyverno Installation

```bash
# Check Kyverno pods
kubectl get pods -n kyverno

# Check Kyverno CRDs
kubectl get crds | grep kyverno

# List available policy types
echo "Kyverno Policy CRDs:"
kubectl api-resources | grep kyverno
```

Expected output (you should see resources like):
```
clusterpolicies                     cpol                kyverno.io/v1                     false        ClusterPolicy
policies                            pol                 kyverno.io/v1                     true         Policy
policyexceptions                    polex               kyverno.io/v2                     true         PolicyException
validatingpolicies                  vpol                policies.kyverno.io/v1beta1       false        ValidatingPolicy
mutatingpolicies                    mpol                policies.kyverno.io/v1beta1       false        MutatingPolicy
ephemeralreports                    ephr                reports.kyverno.io/v1             true         EphemeralReport
```

> **Note**: The exact CRDs may vary depending on your Kyverno version. The key resources are `clusterpolicies` and `policies` in the `kyverno.io` API group.

## 📋 Step 3: Create a Test Namespace

```bash
# Create a namespace for policy testing
kubectl create namespace policy-test

echo "✅ Test namespace created"
```

## 🛡️ Step 4: Create a Validation Policy - Require Labels

This policy requires all pods to have specific labels:

```bash
# Create a validation policy
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
  annotations:
    policies.kyverno.io/title: Require Labels
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/description: >-
      Requires all pods to have 'app' and 'owner' labels.
spec:
  validationFailureAction: Enforce
  background: true
  rules:
  - name: require-app-label
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - policy-test
    validate:
      message: "The label 'app' is required on all pods"
      pattern:
        metadata:
          labels:
            app: "?*"
  - name: require-owner-label
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - policy-test
    validate:
      message: "The label 'owner' is required on all pods"
      pattern:
        metadata:
          labels:
            owner: "?*"
EOF

echo "✅ Validation policy 'require-labels' created"
```

> **💡 Tip - Combining Rules**: The policy above uses separate rules for each label. In production, you can combine them into a single rule for simpler management and fewer webhook evaluations:
> ```yaml
> validate:
>   message: "Labels 'app' and 'owner' are required"
>   pattern:
>     metadata:
>       labels:
>         app: "?*"
>         owner: "?*"
> ```

## 🧪 Step 5: Test the Validation Policy

Test with a pod that **violates** the policy:

```bash
# Try to create a pod without required labels
cat <<EOF | kubectl apply -f - 2>&1
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-no-labels
  namespace: policy-test
spec:
  containers:
  - name: nginx
    image: nginx:alpine
EOF

echo "⬆️ The above should show a policy violation error"
```

Expected error:
```
Error from server: error when creating "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request:
The label 'app' is required on all pods
```

Now test with a **compliant** pod:

```bash
# Create a pod with required labels
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-with-labels
  namespace: policy-test
  labels:
    app: my-app
    owner: student-$STUDENT_INITIALS
spec:
  containers:
  - name: nginx
    image: nginx:alpine
EOF

echo "✅ Pod with required labels created successfully"

# Verify the pod is running
kubectl get pods -n policy-test
```

## 🔧 Step 6: Create a Mutation Policy - Add Default Labels

This policy automatically adds labels if they're missing:

```bash
# Create a mutation policy
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-labels
  annotations:
    policies.kyverno.io/title: Add Default Labels
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/description: >-
      Automatically adds default labels to pods if not present.
spec:
  background: false
  rules:
  - name: add-team-label
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - policy-test
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            +(team): "platform-engineering"
            +(environment): "lab"
            +(managed-by): "kyverno"
EOF

echo "✅ Mutation policy 'add-default-labels' created"
```

## 🧪 Step 7: Test the Mutation Policy

```bash
# Create a pod without the team label
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-mutated
  namespace: policy-test
  labels:
    app: mutate-test
    owner: student-$STUDENT_INITIALS
spec:
  containers:
  - name: nginx
    image: nginx:alpine
EOF

echo "✅ Pod created, checking labels..."

# Check if labels were automatically added
kubectl get pod test-pod-mutated -n policy-test -o jsonpath='{.metadata.labels}' | jq

echo "⬆️ Notice the 'team', 'environment', and 'managed-by' labels were added automatically!"
```

## 🚫 Step 8: Create a Policy to Disallow Privileged Containers

```bash
# Create a policy to block privileged containers
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged-containers
  annotations:
    policies.kyverno.io/title: Disallow Privileged Containers
    policies.kyverno.io/category: Pod Security Standards (Baseline)
    policies.kyverno.io/severity: high
    policies.kyverno.io/description: >-
      Privileged containers can access host resources. This policy 
      prevents privileged containers from being created.
spec:
  validationFailureAction: Enforce
  background: true
  rules:
  - name: privileged-containers
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - policy-test
    validate:
      message: >-
        Privileged containers are not allowed. 
        Set securityContext.privileged to false.
      pattern:
        spec:
          containers:
          - =(securityContext):
              =(privileged): "false"
EOF

echo "✅ Policy 'disallow-privileged-containers' created"
```

> **⚠️ Production Note**: This example focuses on `containers` for brevity. In production environments, you should also validate `initContainers` and `ephemeralContainers` to prevent privilege escalation through those vectors:
> ```yaml
> pattern:
>   spec:
>     =(containers):
>     - =(securityContext):
>         =(privileged): false
>     =(initContainers):
>     - =(securityContext):
>         =(privileged): false
> ```

## 🧪 Step 9: Test the Privileged Container Policy

```bash
# Try to create a privileged container
cat <<EOF | kubectl apply -f - 2>&1
apiVersion: v1
kind: Pod
metadata:
  name: test-privileged-pod
  namespace: policy-test
  labels:
    app: privileged-test
    owner: student-$STUDENT_INITIALS
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    securityContext:
      privileged: true
EOF

echo "⬆️ The above should show a policy violation error"
```

## 📊 Step 10: View Policy Reports

Kyverno generates reports for policy results. Reports are scoped based on the resources being evaluated:
- **`ephemeralreports`** - Temporary reports for namespaced resources (Pods, Deployments, etc.)
- **`clusterephemeralreports`** - Temporary reports for cluster-scoped resources (Namespaces, ClusterRoles, etc.)
- **`policyreports`** - Aggregated, persistent reports for namespaced resources
- **`clusterpolicyreports`** - Aggregated, persistent reports for cluster-scoped resources

> **💡 Ephemeral vs PolicyReports**: Ephemeral reports are short-lived (a few minutes) and get aggregated into PolicyReports, then cleaned up to save storage. If you don't see ephemeral reports, check the aggregated PolicyReports instead.

```bash
# View aggregated policy reports (persistent - check these first!)
kubectl get policyreports -n policy-test
kubectl get clusterpolicyreports -A

# View ephemeral reports (temporary - may be empty if already aggregated)
kubectl get ephemeralreports -n policy-test
kubectl get ephemeralreports -A

# Alternative: Check policy status directly on the ClusterPolicy
kubectl get clusterpolicy -o custom-columns=NAME:.metadata.name,READY:.status.ready,MESSAGE:.status.conditions[0].message

# View detailed policy status
kubectl describe clusterpolicy require-labels | grep -A 10 "Status:"
```

> **Note**: Policy reports may show "No resources found" if:
> - No violations have occurred (policies in `Enforce` mode block resources before they're created)
> - Background scanning hasn't completed yet
> - You're checking the wrong report type (use `policyreports` for Pods, not `clusterpolicyreports`)

**Understanding Policy Modes and Reports:**

| Mode            | Violation Blocked?       | Report Generated?                          |
| --------------- | ------------------------ | ------------------------------------------ |
| `Enforce`       | ✅ Resource blocked       | ❌ No (resource never created)              |
| `Audit`         | ❌ Resource allowed       | ✅ PolicyReport for violation               |
| Background scan | N/A (existing resources) | ✅ PolicyReport for non-compliant resources |

> **💡 Why "No resources found"?** Since our policies use `Enforce` mode, violations are blocked at admission time and never reach etcd. No resource = no report. You'll see reports when using `Audit` mode (Step 11) or when compliant resources are scanned.

## 🏷️ Step 11: Require Resource Requests and Limits

This policy validates that resource requests and limits are set (note: this is a validation policy, not a Kubernetes `ResourceQuota`):

```bash
# Create resource requirements policy
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
  annotations:
    policies.kyverno.io/title: Require Resource Limits
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/severity: medium
spec:
  validationFailureAction: Audit
  background: true
  rules:
  - name: validate-resources
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - policy-test
    validate:
      message: "CPU and memory resource requests and limits are required"
      pattern:
        spec:
          containers:
          - resources:
              limits:
                memory: "?*"
                cpu: "?*"
              requests:
                memory: "?*"
                cpu: "?*"
EOF

echo "✅ Policy 'require-resource-limits' created (Audit mode)"
```

## 📋 Step 12: View All Policies

```bash
# List all cluster policies
kubectl get clusterpolicies

# Get policy details
kubectl describe clusterpolicy require-labels

# Check policy status
kubectl get clusterpolicies -o custom-columns=NAME:.metadata.name,READY:.status.ready,BACKGROUND:.spec.background,ACTION:.spec.validationFailureAction
```

## 🧹 Step 13: Cleanup Test Resources

```bash
# Delete test pods
kubectl delete pods --all -n policy-test

# Keep the policies for comparison with OPA in the next chapter
echo "✅ Test pods cleaned up (policies retained)"
```

## 🎓 Summary

You have successfully:
- ✅ Installed Kyverno on your cluster
- ✅ Created validation policies to enforce standards
- ✅ Created mutation policies to automatically modify resources
- ✅ Tested policy enforcement with real scenarios
- ✅ Learned about policy reports for auditing

## 📝 Important Notes

- **validationFailureAction**: Use `Enforce` to block violations, `Audit` to log only
- **background**: When true, policies scan existing resources
- **Mutation order**: Mutations happen before validations
- **Policy scope**: Use `ClusterPolicy` for cluster-wide, `Policy` for namespace-scoped

## 🔍 Troubleshooting

```bash
# Check Kyverno logs
kubectl logs -n kyverno -l app.kubernetes.io/name=kyverno -f

# Check if webhook is registered
kubectl get validatingwebhookconfigurations | grep kyverno
kubectl get mutatingwebhookconfigurations | grep kyverno

# Describe a policy for status
kubectl describe clusterpolicy require-labels
```

## � Enterprise Preview: PolicyExceptions

In enterprise environments, you often need to exempt specific workloads from policies (e.g., system components, legacy apps). Kyverno supports **PolicyException** resources for this:

```yaml
# Example - NOT for this lab
apiVersion: kyverno.io/v2beta1
kind: PolicyException
metadata:
  name: allow-privileged-monitoring
  namespace: kyverno
spec:
  exceptions:
  - policyName: disallow-privileged-containers
    ruleNames:
    - validate-privileged
  match:
    any:
    - resources:
        namespaces:
        - monitoring
        names:
        - prometheus-*
```

> **Note**: PolicyExceptions require additional RBAC controls and are often restricted to cluster admins or GitOps workflows.

## �🚀 Next Steps

Continue to **[Chapter 2: Policy Engines - OPA Gatekeeper & Rego](../chapter-2-opa-gatekeeper/README.md)** to learn about an alternative policy engine using Rego.

---

**Questions or Issues?**
- Check Kyverno pod logs for errors
- Verify webhook configurations are present
- Ensure namespace matches policy rules
- Ask your instructor for assistance

---

## ⚠️ Disclaimer

Educational/lab purposes only. Calculations and/or statements may contain errors.

This documentation is provided "as is" without warranty of any kind. The author takes no responsibility for any errors, omissions, or inaccuracies contained herein. This material may contain incorrect or outdated information. Always verify configurations against official Microsoft Azure and Kubernetes documentation before use in production environments. Use at your own risk.
