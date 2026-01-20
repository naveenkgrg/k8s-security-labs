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
kubectl config set-context --current --namespace=lab-serviceaccount #optional it will reduce burden to give namespace in each command.
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

curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" $APISERVER/api/v1/namespaces/lab-serviceaccount/pods

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
Verify in pod by running following command you should able to see api success with following commands:
```bash
curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" $APISERVER/api/v1/namespaces/lab-serviceaccount/pods
```

**The difference:**
Same token, same authentication, but now RBAC grants authorization.

---

### Step 12: Cleanup - Return to Baseline

To clean up all resources and return your cluster to its baseline state, follow these steps:

**Step 12a: Delete the Kubernetes namespace**

This removes all Kubernetes resources created during the lab (Pods, ServiceAccounts, Roles, RoleBindings):


**Option 2 - Faster deletion (immediate, recommended):**
```bash
kubectl delete ns lab-serviceaccount --grace-period=0 --force
```

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

**Lab Version:** 1.0  
**Educational Purpose** – Free to use and modify

