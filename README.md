# K3s GitOps HomeLab

An educational showcase repository featuring a complete GitOps setup running on a K3s Kubernetes cluster with a Raspberry Pi 5 control plane and two Intel NUC8i7 worker nodes.

## 🏗️ Architecture

This project demonstrates a modern Kubernetes deployment pipeline featuring:
- **K3s**: Lightweight Kubernetes distribution running on mixed ARM/x86 architecture
- **Cilium**: eBPF-based CNI for advanced networking and observability
- **GitOps**: Infrastructure and application management through Git
- **Flux**: GitOps operator for continuous delivery
- ~~**SOPS/AGE**: Secrets encryption and management~~
- **External Secrets Operator**: Secrets management with HashiCorp Vault integration
- **CloudNative PostgreSQL**: Database operator for PostgreSQL workloads
- **Monitoring**: Grafana and kube-prometheus-stack for observability
- **TLS Management**: Cert-manager with Let's Encrypt certificates
- **Internet Exposure**: Cloudflare Tunnels for secure external access
- **Automated Updates**: Renovate for automatic application updates
- **Dev Containers**: Consistent development environment

## 🚀 Features

- **Multi-node hybrid cluster**: Raspberry Pi 5 control plane with Intel NUC worker nodes
- **Advanced networking**: Cilium CNI with eBPF for high-performance networking
- **GitOps workflow**: All configurations managed through Git
- **Encrypted secrets**: External Secrets Operator with HashiCorp Vault for secure secret management
- **Database management**: PostgreSQL clusters via CloudNative-PG operator
- **Complete observability**: Grafana dashboards with Prometheus metrics
- **Automated TLS**: Let's Encrypt certificates via cert-manager
- **Zero-config internet access**: Cloudflare Tunnels without firewall changes
- **Automatic dependency updates**: Renovate bot for keeping applications current
- **Development ready**: Preconfigured dev container for immediate productivity

## 🛠️ Tech Stack

- **Kubernetes**: K3s v1.33.4
- **CNI**: Cilium (eBPF-based)
- **GitOps**: Flux v2
- **Secrets**: External Secrets Operator + HashiCorp Vault
- **Database**: CloudNative PostgreSQL Operator
- **Monitoring**: Grafana + Prometheus (kube-prometheus-stack)
- **TLS**: cert-manager + Let's Encrypt
- **Networking**: Cloudflare Tunnels
- **Updates**: Renovate bot
- **Development**: Dev Container
- **Hardware**: Raspberry Pi 5 (ARM) + Intel NUC8i7 (x86)

## 🎯 Purpose

This repository serves as a comprehensive example for:
- Learning GitOps principles and practices
- Understanding Kubernetes operations on hybrid edge hardware
- Implementing eBPF-based networking with Cilium
- Implementing secure secret management workflows
- Exploring cloud-native database operations
- Setting up complete monitoring and observability stack
- Managing TLS certificates automatically
- Exposing services to the internet securely without network configuration
- Automating dependency management with Renovate
- Setting up reproducible development environments

## 🔐 Security

Sensitive data is securely stored in HashiCorp Vault and dynamically synchronized to the cluster via External Secrets Operator, ensuring robust security while maintaining GitOps best practices. Additionally, Cloudflare Tunnels provide secure internet exposure without opening firewall ports or exposing the home network, creating an encrypted connection between the cluster and Cloudflare's edge network.

---

*This is a learning project showcasing modern cloud-native practices on edge computing hardware.*
