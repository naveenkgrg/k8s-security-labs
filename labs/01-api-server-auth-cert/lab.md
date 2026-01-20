# Lab 01: Inspecting API Server Authentication Certificates

## Objectives

Extract and inspect the client certificate from your kubeconfig to understand how Kubernetes authenticates users via X.509 certificates.

## Duration

Approximately 15-20 minutes

## Prerequisites

- Kubernetes cluster (Minikube, Docker Desktop, or cloud-hosted)
- `kubectl`, `openssl`, `base64` command-line tools
- Basic shell familiarity

## Steps

### 1. Display Your kubeconfig File

```bash
kubectl config view
```

This shows the cluster configuration including user authentication credentials.

### 2. Extract the Client Certificate

```bash
CERT_DATA=$(kubectl config view --raw -o jsonpath='{.users[0].user.client-certificate-data}')
echo "${CERT_DATA:0:100}"
```

This extracts the base64-encoded certificate from the kubeconfig file using jsonpath.

### 3. Decode the Certificate

```bash
CERT_DATA=$(kubectl config view --raw -o jsonpath='{.users[0].user.client-certificate-data}')
echo "$CERT_DATA" | base64 -d > /tmp/client-cert.pem
file /tmp/client-cert.pem
cat /tmp/client-cert.pem | head -3
```

This decodes the base64 string and saves the certificate in PEM format.

### 4. Inspect the Certificate

```bash
openssl x509 -in /tmp/client-cert.pem -text -noout
```

This displays the certificate structure including version, serial number, issuer, validity dates, subject fields, and public key information.

### 5. Extract the Common Name (Username)

```bash
openssl x509 -in /tmp/client-cert.pem -noout -subject
openssl x509 -in /tmp/client-cert.pem -noout -subject | grep -o 'CN[^,]*'
```

The CN field maps to the Kubernetes username used for authorization checks.

### 6. Extract Organizations (Groups)

```bash
openssl x509 -in /tmp/client-cert.pem -noout -subject
openssl x509 -in /tmp/client-cert.pem -noout -subject | grep -o 'O[^,]*'
```

The O field(s) map to Kubernetes group memberships. Multiple O fields indicate multiple group memberships.

### 7. View All Subject Fields

```bash
openssl x509 -in /tmp/client-cert.pem -noout -subject
openssl x509 -in /tmp/client-cert.pem -noout -subject | sed 's/subject=//' | tr ',' '\n'
```

This displays all identity fields from the certificate subject.

### 8. Check Certificate Validity Dates

```bash
openssl x509 -in /tmp/client-cert.pem -noout -dates
```

Certificates have expiration dates. When expired, authentication will fail.

### 9. Verify the Certificate Chain

```bash
CA_DATA=$(kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')
echo "$CA_DATA" | base64 -d > /tmp/ca-cert.pem
openssl verify -CAfile /tmp/ca-cert.pem /tmp/client-cert.pem
```

This verifies that your client certificate was signed by the cluster's Certificate Authority.

### 10. Extract Issuer Information

```bash
openssl x509 -in /tmp/client-cert.pem -noout -issuer
```

The issuer is the Certificate Authority that signed your certificate.

### 11. View Your Kubernetes Identity

```bash
USERNAME=$(openssl x509 -in /tmp/client-cert.pem -noout -subject | grep -o 'CN[^,]*' | cut -d= -f2)
GROUP=$(openssl x509 -in /tmp/client-cert.pem -noout -subject | grep -o 'O[^,]*' | cut -d= -f2)
echo "Username: $USERNAME"
echo "Group: $GROUP"
```

This is how Kubernetes identifies you when making API requests.

### 12. Test Certificate-Based Authentication

```bash
kubectl get nodes
kubectl auth can-i get nodes
```

Verify that your certificate-based authentication is working.

## Key Takeaways

- **CN (Common Name)**: Maps to the Kubernetes username used for RBAC authorization
- **O (Organization)**: Maps to group memberships
- **Issuer**: The Certificate Authority that signed your certificate
- **Validity**: Certificates have expiration dates; expired certificates prevent authentication
- **Trust Chain**: The API server trusts certificates signed by its Certificate Authority

## Cleanup

```bash
rm -f /tmp/client-cert.pem /tmp/ca-cert.pem
```

## References

- [Kubernetes Authentication Documentation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [X.509 Certificates in Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/certificates/)
- [kubeconfig Format](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
- [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
