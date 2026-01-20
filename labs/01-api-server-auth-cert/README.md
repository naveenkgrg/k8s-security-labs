# Lab 01: API Server Authentication Using Client Certificates

## Overview

This lab explores certificate-based authentication in Kubernetes. Every request to the API server must be authenticated. In this case, `kubectl` presents a client certificate to prove its identity. Understanding the structure and content of these certificates is essential for debugging authentication issues, managing cluster access, and implementing proper security policies.

## Key Concepts

The Kubernetes API server enforces authentication before processing any request. Certificate-based authentication works by extracting identity information from the certificate presented by the client:

- **Common Name (CN)**: Maps to the Kubernetes username
- **Organization (O)**: Maps to group membership

The certificate is verified against the cluster's Certificate Authority before the identity is extracted.

## Prerequisites

A running Kubernetes cluster and these command-line tools:
- `kubectl`
- `openssl`
- `base64`

Verify your setup:
```bash
kubectl version --client
openssl version
```

## Lab Duration

Approximately 15-20 minutes for the complete exercise.

## What You'll Do

Follow the hands-on steps in [lab.md](./lab.md) to:
- Extract your client certificate from the kubeconfig file
- Decode and inspect the X.509 certificate structure
- Identify the username and group information
- Verify the certificate chain and issuer
- Understand how these values are used by the API server

## What You'll Learn

- How to read and interpret X.509 certificates
- The relationship between certificate fields and Kubernetes identity
- How to verify certificate validity and trust chain
- Practical troubleshooting techniques for authentication issues

## Frequently Asked Questions

**Do I need a special cluster?** Any Kubernetes cluster works - local or cloud-hosted.

**Does this modify the cluster?** No. This lab only reads and inspects existing authentication data.

**How does this relate to RBAC?** Authentication determines *who* you are. RBAC (covered in Lab 02) determines what you're allowed to do.

**Can I reuse this material?** Yes. Feel free to adapt and share.

## References

- [Kubernetes Authentication Documentation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [X.509 Certificates in Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/certificates/)
- [kubeconfig Format](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
- [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

