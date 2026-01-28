# Chapter 2: Policy Engines - OPA Gatekeeper & Rego

In this chapter, you'll learn how to enforce policies using OPA Gatekeeper and the Rego policy language. This provides an alternative approach to Kyverno with more powerful policy logic capabilities.

## 🎯 Learning Objectives

- Understand Open Policy Agent (OPA) and Gatekeeper
- Install and configure Gatekeeper
- Write policies using the Rego language
- Create ConstraintTemplates and Constraints
- Compare Gatekeeper with Kyverno

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- Completed [Chapter 1: Policy Engines - Kyverno](../chapter-1-kyverno/README.md)
- AKS cluster running

## 🔄 Load Your Configuration

```bash
# Load your lab configuration
source ./lab-config.sh

# Verify cluster access
kubectl get nodes
```

## 📖 Understanding OPA Gatekeeper

**Open Policy Agent (OPA)** is a general-purpose policy engine that uses the Rego language. **Gatekeeper** is the Kubernetes-native implementation of OPA.

Key concepts:
- **Rego**: A declarative query language for writing policies
- **ConstraintTemplate**: Defines the policy logic using Rego
- **Constraint**: Applies the template with specific parameters
- **Audit**: Gatekeeper periodically audits existing resources (default: every 60 seconds)

> **🧠 Mental Model: Kyverno vs Gatekeeper**
> - **Kyverno**: "Kubernetes-native guardrails with YAML" — great for teams who want to stay in the Kubernetes ecosystem
> - **Gatekeeper**: "General-purpose policy engine with Kubernetes adapters" — ideal when you need complex logic or use OPA elsewhere
>
> Choose based on your team's existing skills and policy complexity needs.

```
                    ┌────────────────────────────────────────┐
                    │           Kubernetes API Server        │
                    │                                        │
   kubectl apply   │  ┌──────────────────────────────────┐  │
   ──────────────► │  │      Admission Webhook           │  │
                   │  │                                   │  │
                   │  │  ┌─────────────────────────────┐ │  │
                   │  │  │      OPA Gatekeeper         │ │  │
                   │  │  │                             │ │  │
                   │  │  │  ┌───────────────────────┐  │ │  │
                   │  │  │  │   Rego Policy Engine  │  │ │  │
                   │  │  │  └───────────────────────┘  │ │  │
                   │  │  │            ▲                │ │  │
                   │  │  │  ┌─────────┴─────────────┐  │ │  │
                   │  │  │  │ConstraintTemplates   │  │ │  │
                   │  │  │  │    + Constraints      │  │ │  │
                   │  │  │  └───────────────────────┘  │ │  │
                   │  │  └─────────────────────────────┘ │  │
                   │  └──────────────────────────────────┘  │
                   └────────────────────────────────────────┘
```

## 📦 Step 1: Install Gatekeeper

```bash
# Create namespace for Gatekeeper
kubectl create namespace gatekeeper-system

# Install Gatekeeper using Helm
helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system \
  --set replicas=1 \
  --set auditInterval=60

echo "⏳ Waiting for Gatekeeper to be ready..."
kubectl wait --for=condition=ready pod -l control-plane=controller-manager -n gatekeeper-system --timeout=300s

echo "✅ Gatekeeper installed successfully"
```

## 🔍 Step 2: Verify Gatekeeper Installation

```bash
# Check Gatekeeper pods
kubectl get pods -n gatekeeper-system

# Check Gatekeeper CRDs
kubectl get crds | grep gatekeeper

# Check constraint templates (should be empty initially)
kubectl get constrainttemplates
```

Expected pods:
```
gatekeeper-audit-xxxxx                1/1     Running
gatekeeper-controller-manager-xxxxx   1/1     Running
```

Expected CRDs (you should see many, including):
```
configs.config.gatekeeper.sh
constrainttemplates.templates.gatekeeper.sh
assign.mutations.gatekeeper.sh
assignmetadata.mutations.gatekeeper.sh
providers.externaldata.gatekeeper.sh
```

> **Note**: The `kubectl get constrainttemplates` command will show "No resources found" initially - this is expected! We will create ConstraintTemplates in the following steps.

> **📦 Enterprise Pattern**: ConstraintTemplates are cluster-scoped and typically managed by platform teams (often via GitOps). Constraints are how application teams enable those policies for their namespaces.

## 📋 Step 3: Create a Test Namespace

```bash
# Create a namespace for Gatekeeper testing (different from Kyverno)
kubectl create namespace gatekeeper-test

echo "✅ Test namespace created"
```

## 🛡️ Step 4: Create Your First ConstraintTemplate

This template defines a policy that requires specific labels:

```bash
# Create a ConstraintTemplate for required labels
cat <<EOF | kubectl apply -f -
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
  annotations:
    description: "Requires specified labels on resources"
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              description: "List of required labels"
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        # Deny if any required label is missing
        violation[{"msg": msg}] {
          # Get the list of required labels from parameters
          required := input.parameters.labels[_]
          
          # Check if the label exists on the resource
          not input.review.object.metadata.labels[required]
          
          # Create the violation message
          msg := sprintf("Resource must have label: %v", [required])
        }
EOF

echo "✅ ConstraintTemplate 'k8srequiredlabels' created"
```

## 📝 Step 5: Understanding Rego

Let's break down the Rego policy:

```rego
package k8srequiredlabels          # Package name

violation[{"msg": msg}] {          # Rule that creates violations
  required := input.parameters.labels[_]    # Iterate over required labels
  not input.review.object.metadata.labels[required]  # Check if missing
  msg := sprintf("Resource must have label: %v", [required])
}
```

Key Rego concepts:
- **input.review.object**: The Kubernetes resource being evaluated
- **input.parameters**: Values passed from the Constraint
- **violation**: Set of violations found
- **:=**: Assignment operator
- **[_]**: Iterate over array elements

## 🎯 Step 6: Create a Constraint

Apply the template to enforce the policy:

```bash
# Create a Constraint using the template
cat <<EOF | kubectl apply -f -
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: pods-must-have-labels
spec:
  enforcementAction: deny
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces:
    - gatekeeper-test
  parameters:
    labels:
    - "app"
    - "owner"
EOF

echo "✅ Constraint 'pods-must-have-labels' created"
```

## 🧪 Step 7: Test the Required Labels Policy

Test with a pod that **violates** the policy:

```bash
# Try to create a pod without required labels
cat <<EOF | kubectl apply -f - 2>&1
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-no-labels
  namespace: gatekeeper-test
spec:
  containers:
  - name: nginx
    image: nginx:alpine
EOF

echo "⬆️ The above should show a policy violation error"
```

Expected error:
```
Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request: 
[pods-must-have-labels] Resource must have label: app
[pods-must-have-labels] Resource must have label: owner
```

Now test with a **compliant** pod:

```bash
# Create a pod with required labels
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-with-labels
  namespace: gatekeeper-test
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
kubectl get pods -n gatekeeper-test
```

## 🚫 Step 8: Create a Template for Container Limits (Presence Check)

This template validates that containers have resource limits defined.

> **⚠️ Important**: Comparing resource quantities (e.g., `250m` vs `500m`) requires parsing Kubernetes quantity strings, which is non-trivial in Rego. This example validates **presence** of limits, not numeric comparison. For production numeric validation, consider using Gatekeeper's external data feature or pre-processing.

```bash
# Create a ConstraintTemplate for container limits
cat <<EOF | kubectl apply -f -
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8scontainerlimits
  annotations:
    description: "Requires containers to have resource limits (presence check)"
spec:
  crd:
    spec:
      names:
        kind: K8sContainerLimits
      validation:
        openAPIV3Schema:
          type: object
          properties:
            cpu:
              type: string
              description: "Maximum CPU limit"
            memory:
              type: string
              description: "Maximum memory limit"
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8scontainerlimits

        # Deny if container lacks CPU limits
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container '%v' must have CPU limits", [container.name])
        }

        # Deny if container lacks memory limits
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container '%v' must have memory limits", [container.name])
        }

        # Deny if CPU limit exceeds maximum
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          cpu_limit := container.resources.limits.cpu
          max_cpu := input.parameters.cpu
          not is_cpu_within_limit(cpu_limit, max_cpu)
          msg := sprintf("Container '%v' CPU limit '%v' exceeds maximum '%v'", [container.name, cpu_limit, max_cpu])
        }

        # Helper function - validates presence only, not numeric comparison
        # Note: True quantity comparison requires parsing K8s quantity strings
        is_cpu_within_limit(limit, max) {
          limit != ""
        }
EOF

echo "✅ ConstraintTemplate 'k8scontainerlimits' created"
```

## 🎯 Step 9: Apply the Container Limits Constraint

```bash
# Create the Constraint
cat <<EOF | kubectl apply -f -
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sContainerLimits
metadata:
  name: container-must-have-limits
spec:
  enforcementAction: deny
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces:
    - gatekeeper-test
  parameters:
    cpu: "500m"
    memory: "512Mi"
EOF

echo "✅ Constraint 'container-must-have-limits' created"
```

## 🧪 Step 10: Test the Container Limits Policy

```bash
# Try to create a pod without resource limits
cat <<EOF | kubectl apply -f - 2>&1
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-no-limits
  namespace: gatekeeper-test
  labels:
    app: limit-test
    owner: student-$STUDENT_INITIALS
spec:
  containers:
  - name: nginx
    image: nginx:alpine
EOF

echo "⬆️ The above should show a policy violation error"
```

Now create a compliant pod:

```bash
# Create a pod with resource limits
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-with-limits
  namespace: gatekeeper-test
  labels:
    app: limit-test
    owner: student-$STUDENT_INITIALS
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    resources:
      limits:
        cpu: "250m"
        memory: "256Mi"
      requests:
        cpu: "100m"
        memory: "128Mi"
EOF

echo "✅ Pod with resource limits created successfully"
```

## 🔒 Step 11: Create a Template for Privileged Containers

```bash
# Create a ConstraintTemplate to block privileged containers
cat <<EOF | kubectl apply -f -
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sdisallowprivileged
  annotations:
    description: "Disallows privileged containers"
spec:
  crd:
    spec:
      names:
        kind: K8sDisallowPrivileged
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdisallowprivileged

        # Deny privileged containers
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("Privileged containers are not allowed: '%v'", [container.name])
        }

        # Also check init containers
        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          container.securityContext.privileged == true
          msg := sprintf("Privileged init containers are not allowed: '%v'", [container.name])
        }
EOF

echo "✅ ConstraintTemplate 'k8sdisallowprivileged' created"
```

> **📝 Scope Note**: This policy covers `containers` and `initContainers`. It does **not** cover `ephemeralContainers` (used for debugging). For comprehensive pod security, consider using Kubernetes Pod Security Standards (PSS) or Pod Security Admission (PSA) alongside Gatekeeper.

## 🎯 Step 12: Apply the Privileged Container Constraint

```bash
# Create the Constraint
cat <<EOF | kubectl apply -f -
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowPrivileged
metadata:
  name: no-privileged-containers
spec:
  enforcementAction: deny
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces:
    - gatekeeper-test
EOF

echo "✅ Constraint 'no-privileged-containers' created"
```

## 📊 Step 13: View Audit Results

Gatekeeper periodically audits existing resources against your policies:

> **⏱️ Timing**: Audit violations are **eventually consistent** and may take up to one audit interval (default: 60 seconds) to appear. This differs from admission-time enforcement, which is immediate.

```bash
# Check constraint status and violations
kubectl get constraints

# Get detailed violation information
kubectl describe k8srequiredlabels pods-must-have-labels

# View all constraint violations
kubectl get constraints -o json | jq '.items[].status.violations'
```

## 📋 Step 14: Compare Kyverno vs Gatekeeper

Let's review both approaches:

```bash
echo "========================================"
echo "  POLICY ENGINE COMPARISON"
echo "========================================"
echo ""
echo "KYVERNO:"
echo "  - Uses native Kubernetes YAML"
echo "  - Easier to learn for K8s users"
echo "  - Built-in mutation support"
echo "  - Policy generation capabilities"
kubectl get clusterpolicies
echo ""
echo "GATEKEEPER:"
echo "  - Uses Rego policy language"
echo "  - More powerful logic capabilities"
echo "  - Template + Constraint pattern"
echo "  - General-purpose policy engine"
kubectl get constrainttemplates
kubectl get constraints
```

| Feature             | Kyverno          | OPA Gatekeeper     |
| ------------------- | ---------------- | ------------------ |
| Policy Language     | YAML             | Rego               |
| Learning Curve      | Low              | Medium-High        |
| Mutation Support    | Built-in         | External (Gator)   |
| Policy Generation   | Yes              | No                 |
| Logic Complexity    | Basic            | Advanced           |
| Community Templates | Kyverno Policies | Gatekeeper Library |

## 🧹 Step 15: Cleanup Test Resources

```bash
# Delete test pods
kubectl delete pods --all -n gatekeeper-test

# Optionally remove constraints and templates
# kubectl delete constraints --all
# kubectl delete constrainttemplates --all

echo "✅ Test pods cleaned up"
```

## 🎓 Summary

You have successfully:
- ✅ Installed OPA Gatekeeper on your cluster
- ✅ Learned the Rego policy language basics
- ✅ Created ConstraintTemplates with Rego policies
- ✅ Applied Constraints to enforce policies
- ✅ Compared Gatekeeper with Kyverno

## 📝 Important Notes

**enforcementAction Reference:**

| Action   | Effect                           |
| -------- | -------------------------------- |
| `deny`   | Block resource creation/update   |
| `warn`   | Allow resource + emit warning    |
| `dryrun` | Audit only (no admission impact) |

- **Template vs Constraint**: Templates define logic, Constraints apply them with parameters
- **Rego debugging**: Use the Rego Playground (https://play.openpolicyagent.org/)
- **Policy Library**: Check https://open-policy-agent.github.io/gatekeeper-library/

## 🔍 Troubleshooting

```bash
# Check Gatekeeper logs
kubectl logs -n gatekeeper-system -l control-plane=controller-manager -f

# Check if webhook is registered
kubectl get validatingwebhookconfigurations | grep gatekeeper

# Debug constraint status
kubectl describe constrainttemplate k8srequiredlabels

# Test Rego policy locally
# Use: https://play.openpolicyagent.org/
```

## 🏢 Enterprise Considerations

**Policy Exceptions**: Unlike Kyverno's native `PolicyException` resource, Gatekeeper handles exceptions through:
- Namespace exclusions in Constraint `match` blocks
- Label-based exclusions in Rego logic
- Config resource for system namespace exclusions

For break-glass scenarios, teams typically use CI/CD overrides or temporary constraint deletion (with audit trails).

**External Data**: Gatekeeper can query external systems using `ExternalData` providers. This enables policies that check against external registries, CMDBs, or approval systems — a key differentiator for complex enterprise scenarios.

## 🚀 Next Steps

Continue to **[Chapter 3: Azure Service Operator v2](../chapter-3-aso/README.md)** to learn about managing Azure resources from Kubernetes.

---

**Questions or Issues?**
- Check Gatekeeper pod logs for errors
- Verify ConstraintTemplate syntax is correct
- Use the Rego Playground to test policies
- Ask your instructor for assistance

---

## ⚠️ Disclaimer

Educational/lab purposes only. Calculations and/or statements may contain errors.
