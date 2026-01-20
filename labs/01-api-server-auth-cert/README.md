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


