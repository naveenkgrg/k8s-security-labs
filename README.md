# Kubernetes Security Labs

Hands-on educational labs to understand Kubernetes security concepts and authentication mechanisms.

## Overview

This project contains practical labs focused on Kubernetes security fundamentals. Each lab includes step-by-step instructions and real-world scenarios.

## Labs

- **Lab 01**: API Server Authentication Using Client Certificates
  - Understand X.509 certificate-based authentication
  - Extract and inspect client certificates
  - Learn how identity maps to Kubernetes users and groups

- **Lab 02**: ServiceAccount Tokens and RBAC Authorization
  - Explore Pod authentication via service account tokens
  - Understand JWT token structure
  - Learn role-based access control principles

## Quick Start

Each lab has its own directory with:
- `lab.md` - Step-by-step hands-on instructions
- `README.md` - Conceptual overview and prerequisites

Navigate to a lab directory and start with the README, then follow lab.md.

## Requirements

- Kubernetes cluster (Minikube, Docker Desktop, or cloud-hosted)
- `kubectl` command-line tool
- Basic shell familiarity

## Contributing

Contributions are welcome! Please feel free to:
- Report issues and suggest improvements
- Submit pull requests with enhancements
- Share your feedback and use cases

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

Licensed under MIT License. See [LICENSE](LICENSE) file for details.

## Code of Conduct

Please be respectful and constructive. See [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md).
