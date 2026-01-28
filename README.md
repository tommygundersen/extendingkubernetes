# Extending Kubernetes - Hands-on Lab

Welcome to the Extending Kubernetes Lab! This comprehensive lab will guide you through advanced Kubernetes extension patterns using Policy Engines (Kyverno and OPA/Rego), Cloud Resource Operators (Azure Service Operator v2 and Crossplane), and the Distributed Application Runtime (Dapr).

## 🎯 Learning Objectives

By completing this lab, you will gain hands-on experience with:

- **Policy Enforcement**: Implement admission control policies using Kyverno and OPA/Gatekeeper with Rego
- **Kubernetes-Native Cloud Resources**: Provision Azure resources using Azure Service Operator v2
- **Universal Cloud Control Plane**: Deploy and manage multi-cloud resources using Crossplane with Compositions
- **Microservices Building Blocks**: Implement distributed systems patterns using Dapr

## 📚 Lab Structure

This lab is organized into the following chapters:

- **[Chapter 0: Setup and Prerequisites](./chapter-0-setup/README.md)**
  - Setting your student initials
  - Creating resource group and AKS cluster
  - Verifying cluster access and tools

- **[Chapter 1: Policy Engines - Kyverno](./chapter-1-kyverno/README.md)**
  - Understanding admission controllers
  - Installing Kyverno
  - Creating validation policies
  - Creating mutation policies
  - Testing policy enforcement

- **[Chapter 2: Policy Engines - OPA Gatekeeper & Rego](./chapter-2-opa-gatekeeper/README.md)**
  - Understanding OPA and Gatekeeper
  - Installing Gatekeeper
  - Writing Rego policies
  - Creating ConstraintTemplates
  - Testing policy enforcement

- **[Chapter 3: Azure Service Operator v2](./chapter-3-aso/README.md)**
  - Understanding Kubernetes operators
  - Installing Azure Service Operator v2
  - Creating Azure resources from Kubernetes
  - Managing resource lifecycle
  - Connecting workloads to Azure services

- **[Chapter 4: Crossplane](./chapter-4-crossplane/README.md)**
  - Understanding Crossplane architecture
  - Installing Crossplane
  - Configuring Azure Provider
  - Creating Managed Resources
  - Building Compositions and XRDs

- **[Chapter 5: Dapr Introduction](./chapter-5-dapr/README.md)**
  - Understanding Dapr architecture
  - Installing Dapr on Kubernetes
  - Implementing Service Invocation
  - Using State Management
  - Working with Pub/Sub messaging
  - Exploring Bindings

## ⏱️ Estimated Time

- **Total Lab Duration**: 2-3 hours
- **Chapter 0 (Setup)**: 15-20 minutes
- **Chapter 1 (Kyverno)**: 20-25 minutes
- **Chapter 2 (OPA Gatekeeper)**: 25-30 minutes
- **Chapter 3 (Azure Service Operator v2)**: 25-30 minutes
- **Chapter 4 (Crossplane)**: 30-35 minutes
- **Chapter 5 (Dapr)**: 25-30 minutes

## 🔧 Prerequisites

- Azure subscription with appropriate permissions
- Basic knowledge of Kubernetes concepts (pods, deployments, services, namespaces)
- Familiarity with Azure CLI and kubectl
- Understanding of YAML syntax
- Text editor (VS Code recommended)

## 🚀 Getting Started

1. Start with **[Chapter 0: Setup and Prerequisites](./chapter-0-setup/README.md)**
2. Complete each chapter in sequence
3. Keep your AKS cluster running throughout the lab
4. Each chapter builds on previous concepts

## 💡 Tips for Success

- **Set your initials**: You'll use student initials for resource naming to avoid conflicts
- **Monitor progress**: Use `kubectl get pods -A` to track deployment status
- **Check CRDs**: Use `kubectl get crds` to verify custom resources are installed
- **Save your work**: Keep track of resource names and configurations
- **Ask for help**: Don't hesitate to ask your instructor if you get stuck

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AKS Cluster                                     │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     Policy Engines Layer                             │   │
│  │  ┌─────────────────────┐  ┌─────────────────────────────────────┐   │   │
│  │  │      Kyverno        │  │         OPA Gatekeeper              │   │   │
│  │  │ (Validation/Mutation│  │   (Rego Policies + Constraints)     │   │   │
│  │  └─────────────────────┘  └─────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                   Cloud Resource Operators                           │   │
│  │  ┌─────────────────────┐  ┌─────────────────────────────────────┐   │   │
│  │  │  Azure Service      │  │         Crossplane                  │   │   │
│  │  │  Operator v2        │  │  (Managed Resources + Compositions) │   │   │
│  │  └──────────┬──────────┘  └──────────────────┬──────────────────┘   │   │
│  └─────────────┼────────────────────────────────┼──────────────────────┘   │
│                │                                │                           │
│  ┌─────────────┼────────────────────────────────┼──────────────────────┐   │
│  │             ▼                                ▼                       │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │                    Azure Resources                           │    │   │
│  │  │  Storage Accounts │ Redis Cache │ Service Bus │ etc.        │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Dapr Runtime                                  │   │
│  │  ┌──────────┐ ┌───────────┐ ┌─────────┐ ┌──────────┐ ┌──────────┐   │   │
│  │  │ Service  │ │   State   │ │ Pub/Sub │ │ Bindings │ │  Actors  │   │   │
│  │  │ Invocatn │ │   Store   │ │         │ │          │ │          │   │   │
│  │  └──────────┘ └───────────┘ └─────────┘ └──────────┘ └──────────┘   │   │
│  │                                                                      │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │                   Application Pods                           │    │   │
│  │  │  [App Container] ◄──► [Dapr Sidecar]                        │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 📝 Notes

- All students in the same subscription should use unique initials for naming
- Some resources may take several minutes to provision
- Keep your Azure CLI session active throughout the lab
- Save important commands and outputs for reference

## 🧹 Cleanup

After completing all chapters, refer to the cleanup section in Chapter 0 to remove all resources and avoid unnecessary charges.

---

**Ready to begin?** Head over to **[Chapter 0: Setup and Prerequisites](./chapter-0-setup/README.md)** to get started!

---

## ⚠️ Disclaimer

Educational/lab purposes only. Calculations and/or statements may contain errors.

This documentation is provided "as is" without warranty of any kind. The author takes no responsibility for any errors, omissions, or inaccuracies contained herein. This material may contain incorrect or outdated information. Always verify configurations against official Microsoft Azure and Kubernetes documentation before use in production environments. Use at your own risk.
