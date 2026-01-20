# Lab 02: ServiceAccount Tokens and RBAC Authorization

## Overview

This lab explores how Kubernetes applications authenticate and authorize access to the API server.
Unlike Lab 01 which focused on certificate-based authentication for users, this lab focuses on **ServiceAccount tokens** used by Pods.

You will learn:
- How Pods receive authentication tokens automatically
- How to inspect and use ServiceAccount tokens
- How RBAC controls what authenticated tokens can do
- The interplay between authentication and authorization

## About This Lab

**Lab Objective:** Understand how Pods authenticate to the Kubernetes API server using ServiceAccount tokens and how RBAC controls what those tokens are allowed to do.

**Difficulty Level:** Intermediate ⭐⭐  
**Time Estimate:** 25-30 minutes  
**Format:** Hands-on practical lab  
**Prerequisites:** Basic Kubernetes knowledge; Lab 01 recommended

---

## Core Concepts

### What is a ServiceAccount?

A **ServiceAccount** is a Kubernetes object that provides an identity for applications running inside Pods.

**Key differences from human users:**
- ServiceAccounts are managed by Kubernetes
- Pods authenticate automatically without manual configuration
- Credentials are injected into Pods as files

### Every Namespace Has a Default ServiceAccount

Kubernetes automatically creates a `default` ServiceAccount in every namespace.

If a Pod doesn't specify a ServiceAccount:
- Kubernetes automatically assigns the `default` ServiceAccount
- The default ServiceAccount credentials are mounted into the Pod
- The Pod can use these credentials to communicate with the API server

### ServiceAccount Credentials

A ServiceAccount provides three pieces of information:

1. **Token** – A bearer token for authentication
   - **Kubernetes 1.24+:** Tokens are created dynamically when a Pod uses the ServiceAccount
   - **Note:** When you `describe sa`, it shows `Tokens: none` - this is normal
   - Located at: `/var/run/secrets/kubernetes.io/serviceaccount/token` (inside Pods)
   - Used in HTTP Authorization header: `Authorization: Bearer <token>`

2. **CA Certificate** – To verify the API server
   - Located at: `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`
   - Used for HTTPS/TLS verification

3. **Namespace** – The namespace the Pod is running in
   - Located at: `/var/run/secrets/kubernetes.io/serviceaccount/namespace`
   - Useful for scoped API calls

### How Credentials Are Mounted

When a Pod starts, Kubernetes automatically:
1. Determines which ServiceAccount to use
2. Creates or retrieves the ServiceAccount's token
3. Mounts credentials as files inside the Pod
4. Makes these files available at `/var/run/secrets/kubernetes.io/serviceaccount/`

**This happens automatically** – no manual configuration needed!

### Custom ServiceAccounts

Beyond the `default` ServiceAccount, you can create custom ones:

```bash
kubectl create sa my-app
kubectl create sa data-processor
```

**Benefits:**
- Fine-grained permission control
- Different Pods can have different permissions
- Easier to track which application did what (audit trail)
- Follows principle of least privilege

### Authentication vs. Authorization

These are two different steps:

| Step | Purpose | Mechanism | In This Lab |
|---|---|---|---|
| **Authentication** | Verify who you are | ServiceAccount token | Token is presented to API server |
| **Authorization** | Verify what you can do | RBAC | Roles and RoleBindings grant permissions |

**Important:** A token can be valid (authentication passes) but still not have permission to do something (authorization fails).

### RBAC: Role-Based Access Control

RBAC determines what a ServiceAccount can do:

**Role** – Defines permissions
```
Example: "allowed to get and list Pods"
```

**RoleBinding** – Connects a Role to a ServiceAccount
```
Example: "apply this Role to the 'app-user' ServiceAccount"
```

**The flow:**
1. Pod makes API request with ServiceAccount token
2. API server authenticates the token ✓
3. API server checks RBAC rules
4. If Role + RoleBinding exist, the action is allowed
5. Otherwise, the action is denied

---

## What This Lab Demonstrates

In this practical lab, you will:

1. **Create and inspect ServiceAccounts**
   - See the `default` ServiceAccount
   - Create custom ServiceAccounts
   - Examine ServiceAccount metadata

2. **Run Pods with different ServiceAccounts**
   - Pod using `default` ServiceAccount
   - Pod using custom `app-user` ServiceAccount

3. **Inspect mounted credentials**
   - Examine the files mounted inside Pods
   - Verify the credential location and contents

4. **Call the Kubernetes API**
   - Use the token to authenticate to the API server
   - Understand how applications interact with Kubernetes

5. **Demonstrate authentication vs. authorization**
   - Show how a token can authenticate but lack authorization
   - Use `kubectl auth can-i` to verify permissions

6. **Apply RBAC permissions**
   - Create a Role with specific permissions
   - Create a RoleBinding to grant those permissions
   - Verify the permissions took effect

---

## Prerequisites

### Kubernetes Cluster
Any of the following work:
- Minikube (local development)
- KillerCoda (interactive sandbox) - Recommended
- Docker Desktop with Kubernetes enabled
- Cloud-managed cluster (EKS, GKE, AKS)

## Prerequisites

### Kubernetes Cluster
Any of the following work:
- Minikube (local development)
- KillerCoda (interactive sandbox) - Recommended
- Docker Desktop with Kubernetes enabled
- Cloud-managed cluster (EKS, GKE, AKS)

### Required Tools
- `kubectl` – Kubernetes command-line tool (1.19+)
- `curl` – HTTP client (installed inside Pods as part of lab)
- Basic command-line knowledge

### Knowledge Prerequisites
- Understanding of Kubernetes Pods and namespaces
- Familiarity with `kubectl` commands
- Basic understanding of RBAC concepts helpful (but not required)

**Note:** This lab uses YAML manifests for Pod creation. This is the modern standard approach for specifying ServiceAccounts.

---

## Lab Structure

This lab includes:

- **lab.md** – Step-by-step hands-on instructions
- **README.md** – This file with conceptual background

## How to Use This Lab

1. **Read this README** – Understand the concepts
2. **Follow [lab.md](./lab.md)** – Execute each step
3. **Examine outputs** – See what each step reveals
4. **Understand the flow** – Connect actions to concepts
5. **Clean up** – Remove lab resources

## Expected Outcomes

After completing this lab, you will understand:

✓ How ServiceAccounts provide identity to Pods  
✓ How credentials are automatically injected into Pods  
✓ How Pods use tokens to authenticate to the API server  
✓ The difference between authentication and authorization  
✓ How RBAC controls what authenticated identities can do  
✓ How to create custom ServiceAccounts for fine-grained access control  
✓ How to verify permissions using `kubectl auth can-i`  
✓ The real-world security implications of ServiceAccount and RBAC design  

---

## Security Implications

Understanding this lab helps you:

**Access Control:**
- Design who (which ServiceAccounts) can do what (permissions)
- Implement principle of least privilege in your applications
- Separate concerns between different applications/teams

**Audit & Compliance:**
- Understand how to identify which Pods made API requests
- Track and audit Kubernetes API activity
- Comply with security policies

**Operational Security:**
- Recognize why tokens must be protected
- Understand when to create custom ServiceAccounts
- Know how to troubleshoot authentication failures

**Cloud-Native Best Practices:**
- Use ServiceAccounts instead of static credentials
- Implement proper RBAC policies
- Follow zero-trust principles in your cluster

---

## Real-World Applications

This lab helps you with:

- **Application deployment** – Configuring Pods to have proper permissions
- **Multi-tenant clusters** – Isolating applications with different permissions
- **CI/CD pipelines** – Understanding how deployments interact with Kubernetes
- **Microservices** – Giving each service only the permissions it needs
- **Security audits** – Reviewing and verifying RBAC configurations
- **Troubleshooting** – Diagnosing "permission denied" errors

---

## Educational Purpose

This lab is designed for:

- **Kubernetes learners** seeking practical understanding
- **DevOps engineers** implementing security
- **Platform teams** designing cluster access
- **Security auditors** reviewing Kubernetes configurations
- **Developers** understanding how their Pods interact with Kubernetes
- **Educators** teaching Kubernetes security

---

## Connection to Other Labs

- **Lab 01 (API Server Authentication)** – Covers user certificate authentication
- **Lab 02 (This lab)** – Covers ServiceAccount token authentication and RBAC
- **Lab 03+** – Other security topics (coming soon)

Each lab builds on Kubernetes security concepts progressively.

---

## Common Questions

**Q: Why does my Pod need a ServiceAccount?**  
A: Many applications need to read configuration, watch for changes, or interact with Kubernetes resources. ServiceAccounts provide a secure way to do this.

**Q: Is the default ServiceAccount secure?**  
A: The `default` ServiceAccount has minimal permissions by default. For production apps, use custom ServiceAccounts with specific permissions.

**Q: What happens if I don't specify a ServiceAccount?**  
A: Kubernetes automatically uses the `default` ServiceAccount of that namespace. This is automatic but often not the best security practice.

**Q: Can I change ServiceAccount permissions without recreating Pods?**  
A: Yes! RBAC permissions take effect immediately. No need to restart Pods.

**Q: How do I know which ServiceAccount a Pod is using?**  
A: Use `kubectl describe pod <name>` or check the Pod spec's `serviceAccountName` field.

---

## Next Steps

After completing this lab:

1. **Run comprehensive cleanup** – Follow Step 12 in [lab.md](./lab.md) to return your cluster to baseline state (removes namespace, Pods, RBAC rules, and temporary files)
2. **Review RBAC documentation** – Understand ClusterRoles and ClusterRoleBindings
3. **Explore ServiceAccount tokens** – Learn about token expiration and renewal
4. **Practice RBAC policies** – Create and test various Role configurations
5. **Check other labs** – Explore additional Kubernetes security topics
6. **Implement in your cluster** – Apply what you learned to real workloads

---

## References & Resources

- [Kubernetes ServiceAccounts Documentation](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- [API Access from Pods](https://kubernetes.io/docs/tasks/run-application/access-api-from-pod/)

---

## Lab Information

| Property | Value |
|---|---|
| **Purpose** | Educational understanding of ServiceAccounts and RBAC |
| **Type** | Hands-on practical lab |
| **Difficulty** | Intermediate (builds on Lab 01) |
| **Duration** | 25-30 minutes |
| **Prerequisites** | Kubernetes cluster, kubectl |
| **License** | Educational – Free to use and modify |
| **Contributions** | Welcome – corrections, improvements, alternatives |

---

**Happy Learning!** 🚀

This lab is part of the k8s-security-labs series, designed to build comprehensive Kubernetes security knowledge through hands-on practice.
