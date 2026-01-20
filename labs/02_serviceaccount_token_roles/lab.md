# Lab 02: ServiceAccount Tokens and RBAC Authorization

## Overview

This lab provides hands-on experience with Kubernetes ServiceAccounts and RBAC (Role-Based Access Control).
You will:
- Create and manage ServiceAccounts
- Mount and inspect tokens inside Pods
- Authenticate to the Kubernetes API server using tokens
- Use RBAC to grant and verify permissions
- See how authentication and authorization work together

## Goal

Understand how Pods authenticate to the Kubernetes API server using ServiceAccount tokens and how RBAC controls what those tokens are allowed to do.

## Prerequisites

- Access to a Kubernetes cluster (KillerCoda, Minikube, or cloud cluster)
- `kubectl` command-line tool
- Internet access inside Pods (to install `curl`)
- Basic understanding of Kubernetes Pods and namespaces

**Note:** This lab uses modern Kubernetes (1.24+) which handles ServiceAccount tokens differently than older versions. Tokens are created dynamically when Pods use them, not when ServiceAccounts are created.

## Difficulty Level & Time Estimate

- **Level:** Intermediate ⭐⭐
- **Time:** 25-30 minutes
- **Format:** Hands-on practical lab
- **Prerequisites:** Basic Kubernetes knowledge, Lab 01 recommended

---

## Lab Steps

### Step 0: Create a Namespace

Start fresh by creating a dedicated namespace for this lab:

```bash
kubectl create ns lab-serviceaccount
```

This creates an isolated namespace for our lab experiments.

---

### Step 1: Inspect the Default ServiceAccount

Every namespace in Kubernetes automatically has a `default` ServiceAccount. Let's examine it:

```bash
kubectl -n lab-serviceaccount get sa
kubectl -n lab-serviceaccount describe sa default
```

**What you see:**
- Every namespace has a default ServiceAccount

---

### Step 2: Run a Pod Without Specifying a ServiceAccount

Create a Pod that doesn't explicitly specify a ServiceAccount:

```bash
kubectl -n lab-serviceaccount run alpine-default \
  --image=alpine:3.20 \
  --restart=Never \
  --command -- sh -c "sleep 36000"
```

**Kubernetes behavior:**
When a Pod does not specify a ServiceAccount, Kubernetes automatically assigns the `default` ServiceAccount of that namespace.

---

### Step 3: Verify Which ServiceAccount the Pod Is Using

Check which ServiceAccount is assigned to the Pod:

```bash
kubectl -n lab-serviceaccount describe pod alpine-default | grep -i serviceaccount
```

**Expected output:**
```
Service Account:       default
```

This confirms that the Pod is using the default ServiceAccount.

---

### Step 4: Inspect Mounted ServiceAccount Credentials

Enter the Pod and examine the mounted credentials:

```bash
kubectl -n lab-serviceaccount exec -it alpine-default -- sh
```

Inside the Pod, list the credential files:

```bash
ls /var/run/secrets/kubernetes.io/serviceaccount/
```

**You should see three files:**
- `token` – Used for authentication to the API server
- `ca.crt` – Certificate Authority to trust the API server
- `namespace` – The namespace the Pod is running in
You can check this jwt token with following command..

```bash 
cat /var/run/secrets/kubernetes.io/serviceaccount/token |cut -d. -f2|base64 -d
```

**Why it matters:**
These files are automatically mounted by Kubernetes. Applications inside the Pod can use them to authenticate securely to the Kubernetes API server.

---

### Step 5: Call the Kubernetes API Server Using the Token

Still inside the Pod, set up credentials and test authentication:

```bash
apk add --no-cache curl
```

Set environment variables:

```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
APISERVER=https://kubernetes.default.svc
```

Test authentication by calling the API server's version endpoint:

```bash
curl --cacert $CACERT -H "Authorization: Bearer $TOKEN" $APISERVER/version
```

**Expected result:**
You should see JSON output with version information, confirming authentication succeeded.

---

Try to list Pods in the namespace (this may fail):

```bash
curl --cacert $CACERT -H "Authorization: Bearer $TOKEN" \
  $APISERVER/api/v1/namespaces/lab-serviceaccount/pods
```

**Important distinction:**
- If you see an error like "Forbidden", that means **authentication succeeded but authorization failed**
- The default ServiceAccount token can authenticate, but RBAC denies the request

Exit the Pod:

```bash
exit
```

---

### Step 6: Create a Custom ServiceAccount

Custom ServiceAccounts allow fine-grained permission control. Create one:

```bash
kubectl -n lab-serviceaccount create sa app-user
kubectl -n lab-serviceaccount describe sa app-user
```

**What you see:**
- Your custom ServiceAccount `app-user` is created
- The `Tokens:` field shows `none` (this is normal in Kubernetes 1.24+)

**Important:** In modern Kubernetes, tokens are NOT created when you create a ServiceAccount. Instead:
- Tokens are created dynamically when a Pod uses the ServiceAccount
- The token is mounted directly into the Pod's filesystem
- The token identifies the ServiceAccount as `system:serviceaccount:lab-serviceaccount:app-user`

The token will be available inside the Pod at `/var/run/secrets/kubernetes.io/serviceaccount/token`

---

### Step 7: Run a Pod Using the Custom ServiceAccount

Create a Pod that uses your custom ServiceAccount. Since `kubectl run` doesn't support ServiceAccount specification, we'll create it using a YAML manifest:

First, create a file named `alpine-app.yaml`:

```bash
cat > alpine-app.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: alpine-app
  namespace: lab-serviceaccount
spec:
  serviceAccountName: app-user
  containers:
  - name: alpine
    image: alpine:3.20
    command: ["sh", "-c", "sleep 36000"]
EOF
```

Now create the Pod:

```bash
kubectl apply -f alpine-app.yaml
```

Verify the Pod was created:

```bash
kubectl -n lab-serviceaccount get pods
```

This Pod will use the `app-user` ServiceAccount instead of `default`.

**Why YAML?** Modern kubectl no longer supports specifying ServiceAccount directly in `kubectl run`. Using YAML is the standard approach and provides more control.

---

### Step 8: Verify ServiceAccount Usage and Mounted Credentials

Verify that the Pod is using the correct ServiceAccount:

```bash
kubectl -n lab-serviceaccount describe pod alpine-app | grep -i serviceaccount
```

**Expected output:**
```
Service Account:  app-user
```

Enter the Pod and confirm the credentials are mounted:

```bash
kubectl -n lab-serviceaccount exec -it alpine-app -- sh
ls /var/run/secrets/kubernetes.io/serviceaccount/
exit
```

The credential files are the same, but they represent a different identity (`app-user` instead of `default`).

---

### Step 8a: Extract and Validate the ServiceAccount Token

Now let's extract the actual token from the Pod and validate it:

```bash
kubectl -n lab-serviceaccount exec -it alpine-app -- sh
```

Inside the Pod, extract the token:

```bash
cat /var/run/secrets/kubernetes.io/serviceaccount/token
cat /var/run/secrets/kubernetes.io/serviceaccount/token |cut -d. -f2|base64 -d
```

**Decode and View JWT Token in JSON Format:**

JWT tokens have three parts separated by dots: `header.payload.signature`. Let's decode the payload to see the JSON structure:
**Key Fields to Understand:**

| Field | Meaning | Value in Example |
|---|---|---|
| `sub` | Subject (identity) | `system:serviceaccount:lab-serviceaccount:app-user` |
| `kubernetes.io.namespace` | Namespace | `lab-serviceaccount` |
| `kubernetes.io.serviceaccount.name` | ServiceAccount name | `app-user` |
| `iss` | Issuer (who created token) | Kubernetes API server |
| `iat` | Issued at (timestamp) | Token creation time |
| `exp` | Expiration (timestamp) | When token expires |

This JSON payload contains the **identity information** that Kubernetes uses for authentication and authorization!

### Step 8b: Verify Token Works (Optional - Advanced)

You can also verify the token works by using it to call the Kubernetes API (similar to Step 5):

```bash
kubectl -n lab-serviceaccount exec -it alpine-app -- sh
```

Inside the Pod:

```bash
apk add --no-cache curl

TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
APISERVER=https://kubernetes.default.svc

# Verify token is valid by calling API server
# Note: We use /version (not /api/v1/version) because /version doesn't require RBAC
curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" \
  $APISERVER/version | head -20
```

Exit the Pod:

```bash
exit
```

------

### Step 9: Check Authorization BEFORE Adding RBAC

Before creating any RBAC rules, verify that the `app-user` ServiceAccount has no permissions:

```bash
kubectl -n lab-serviceaccount auth can-i list pods \
  --as=system:serviceaccount:lab-serviceaccount:app-user
```

**Expected output:**
```
no
```

The `app-user` cannot list Pods because no RBAC rules grant permission.

---

### Step 10: Create a Role and RoleBinding

Now grant the `app-user` permission to list Pods by creating a Role:

```bash
kubectl -n lab-serviceaccount create role app-user-reader \
  --verb=get,list,watch \
  --resource=pods
```

Create a RoleBinding to attach the Role to the ServiceAccount:

```bash
kubectl -n lab-serviceaccount create rolebinding app-user-reader-binding \
  --role=app-user-reader \
  --serviceaccount=lab-serviceaccount:app-user
```

**What you created:**
- **Role** – Defines permissions (get, list, watch Pods)
- **RoleBinding** – Connects the Role to the ServiceAccount

---

### Step 11: Verify Authorization AFTER RBAC

Now check if the `app-user` has permission to list Pods:

```bash
kubectl -n lab-serviceaccount auth can-i list pods \
  --as=system:serviceaccount:lab-serviceaccount:app-user
```

**Expected output:**
```
yes
```

**The difference:**
Same token, same authentication, but now RBAC grants authorization.

---

### Step 12: Cleanup - Return to Baseline

To clean up all resources and return your cluster to its baseline state, follow these steps:

**Step 12a: Delete the Kubernetes namespace**

This removes all Kubernetes resources created during the lab (Pods, ServiceAccounts, Roles, RoleBindings):

**Option 1 - Normal deletion (graceful, ~5-10 seconds):**
```bash
kubectl delete ns lab-serviceaccount
```

**Option 2 - Faster deletion (immediate, recommended):**
```bash
kubectl delete ns lab-serviceaccount --grace-period=0 --force
```

The second option is faster because:
- `--grace-period=0` – Skip the 30-second graceful termination period
- `--force` – Force immediate deletion without waiting for confirmation

**Verify the namespace is deleted:**

```bash
kubectl get ns | grep lab-serviceaccount
```

You should see no output, confirming the namespace is deleted.

---

**Step 12b: Remove temporary files**

Remove the YAML manifest file created in Step 7:

```bash
rm -f alpine-app.yaml
```

Verify it's removed:

```bash
ls alpine-app.yaml 2>/dev/null && echo "File still exists" || echo "File removed successfully"
```

---

**Step 12c: Verify clean state**

Confirm your cluster is back to baseline:

```bash
# Check no lab-serviceaccount namespace exists
kubectl get ns | grep lab-serviceaccount || echo "✓ Namespace removed"

# Verify no temp files
ls alpine-app.yaml 2>/dev/null || echo "✓ Temp files cleaned up"

# Show remaining namespaces (should be default ones)
echo "Remaining namespaces:"
kubectl get ns
```

**Expected output:**
- No `lab-serviceaccount` namespace
- No `alpine-app.yaml` file
- Only default namespaces like `default`, `kube-system`, `kube-public`, etc.

---

## Cleanup Summary

Your lab environment is now back to baseline state:

| Resource | Status |
|---|---|
| `lab-serviceaccount` namespace | ✓ Deleted |
| Pods (alpine-default, alpine-app) | ✓ Deleted |
| ServiceAccounts (app-user) | ✓ Deleted |
| Roles and RoleBindings | ✓ Deleted |
| YAML manifest files | ✓ Removed |

---

## Key Concepts Recap

### Authentication vs. Authorization

| Concept | Purpose | In This Lab |
|---|---|---|
| **Authentication** | Proves who you are | ServiceAccount token |
| **Authorization** | Decides what you can do | RBAC Roles and RoleBindings |

### ServiceAccount Tokens

- **Location:** Mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`
- **Purpose:** Used as bearer token in HTTP requests to API server
- **Identity:** Identifies as `system:serviceaccount:<namespace>:<name>`
- **Automatic:** Kubernetes creates and maintains tokens automatically

### RBAC Components

- **Role** – Defines what actions are allowed on which resources
- **RoleBinding** – Connects a Role to a ServiceAccount, Group, or User
- **Permissions flow:** ServiceAccount → RoleBinding → Role → Allowed actions

---

## What You Learned

✓ Pods authenticate using ServiceAccount tokens automatically mounted by Kubernetes  
✓ Every namespace has a default ServiceAccount that any Pod can use  
✓ Custom ServiceAccounts enable fine-grained permission control  
✓ Tokens are mounted as files inside the Pod at a predictable location  
✓ Tokens are JWT (JSON Web Tokens) containing identity information  
✓ You can decode JWT tokens to understand the embedded identity claims  
✓ Tokens represent the identity; RBAC determines permissions  
✓ Authentication proves identity; authorization enforces permissions  
✓ Roles and RoleBindings grant access without changing how tokens work  
✓ `kubectl auth can-i` helps verify permissions before and after RBAC changes  
✓ JWT payload includes namespace, ServiceAccount name, and identity claims  

---

## Next Steps

1. **Explore ClusterRoles** – Understand cluster-wide permissions (not namespace-scoped)
2. **Review RBAC documentation** – Kubernetes official RBAC guide
3. **Experiment with other resources** – Grant permissions for other Kubernetes resources
4. **Combine multiple Roles** – Use multiple RoleBindings for complex permissions
5. **Check Lab 03** – Explore other security topics in this series

---

## Troubleshooting

**Q: `kubectl run --serviceaccount` gives "unknown flag" error?**  
A: Modern kubectl removed this flag. Use YAML manifests instead (see Step 7). Create a YAML file with `serviceAccountName:` field in the Pod spec.

**Q: "Permission denied" when calling the API server?**  
A: This likely means authorization failed (RBAC denied the request). Use Step 9 and Step 11 to verify permissions.

**Q: ServiceAccount token file not found?**  
A: Ensure the Pod has started completely. Wait a moment and try again.

**Q: `kubectl auth can-i` returns unexpected results?**  
A: Verify you're using the correct ServiceAccount name: `system:serviceaccount:lab-serviceaccount:app-user`

**Q: `kubectl describe sa app-user` shows `Tokens: none`?**  
A: This is normal in Kubernetes 1.24+! Tokens are created dynamically when Pods use the ServiceAccount, not stored as Secrets. The token exists inside the running Pod at `/var/run/secrets/kubernetes.io/serviceaccount/token`. Use Step 8a to extract and view the actual token.

**Q: How do I see the actual token value?**  
A: Run Step 8a - exec into the Pod and `cat /var/run/secrets/kubernetes.io/serviceaccount/token`. This shows the JWT token that's mounted in the Pod.

**Q: How do I clean up the Pod created from YAML?**  
A: Either delete the individual Pod with `kubectl -n lab-serviceaccount delete pod alpine-app` or use Step 12 to delete the entire namespace.

---

**Lab Version:** 1.0  
**Educational Purpose** – Free to use and modify

