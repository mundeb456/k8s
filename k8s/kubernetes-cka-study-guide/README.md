# Kubernetes CKA Study Guide

A comprehensive, hands-on study guide for the **Certified Kubernetes Administrator (CKA)** exam, aligned with the **v1.31+** curriculum. This guide is written to be practical and exam-focused: every topic includes real YAML manifests, working `kubectl` commands, and an **Exam Tips** section with the things that trip people up under time pressure.

## About the CKA Exam

The CKA is a **performance-based** certification administered by the Cloud Native Computing Foundation (CNCF) and the Linux Foundation. You do not answer multiple-choice questions — you solve real tasks on live clusters from a terminal.

### Exam Domains and Weightings

| Domain | Weighting |
|--------|-----------|
| Cluster Architecture, Installation & Configuration | **25%** |
| Workloads & Scheduling | **15%** |
| Services & Networking | **20%** |
| Storage | **10%** |
| Troubleshooting | **30%** |

Troubleshooting is the single largest domain — practice reading logs, describing resources, and diagnosing broken clusters until it is second nature.

### Exam Format

- **Duration:** 2 hours
- **Type:** Performance-based (hands-on tasks in a live environment)
- **Environment:** A browser-based terminal with `kubectl` pre-installed; you SSH/`kubectl config use-context` between multiple clusters
- **Number of tasks:** ~15–20 weighted questions
- **Allowed documentation:** During the exam you may open **one** additional browser tab restricted to the official docs:
  - `https://kubernetes.io/docs/` (and its subdomains)
  - `https://kubernetes.io/blog/`
  - `https://helm.sh/docs/`
- **Passing score:** **66%**
- **Retakes:** One free retake included with registration.
- **Validity:** The certification is valid for **3 years** (recently extended from 2).

### What to Master Before Exam Day

- Fluent `kubectl` usage, including imperative commands with `--dry-run=client -o yaml` to generate manifests fast.
- Setting up shell aliases and `kubectl` autocompletion at the start of the exam.
- `etcd` backup and restore.
- Reading cluster component logs (static pods, `journalctl`, `crictl`).

## How to Use This Guide

1. **Work top to bottom.** Topics build on each other — architecture first, then installation, workloads, networking, storage, and finally troubleshooting.
2. **Type every command.** Do not copy-paste. Muscle memory is what saves you time on exam day.
3. **Practice imperatively.** Learn to generate manifests with `kubectl ... --dry-run=client -o yaml > file.yaml` and then edit.
4. **Simulate the clock.** Give yourself ~6 minutes per task in practice.
5. **Bookmark the docs.** Learn where things live on `kubernetes.io` so you can find them in seconds during the exam.

## Prerequisites

- Comfortable on a Linux command line (bash, `vi`/`vim`, `systemctl`, `journalctl`).
- Basic containers/Docker knowledge (images, registries, containers).
- Basic networking (IP, ports, DNS, routing, CIDR).
- A practice cluster: `kubeadm` on 2–3 VMs, `kind`, `minikube`, or a cloud playground like Killercoda or KodeKloud.

## Useful Links

- Official CKA page: <https://www.cncf.io/certification/cka/>
- Exam curriculum (GitHub): <https://github.com/cncf/curriculum>
- Candidate handbook: <https://docs.linuxfoundation.org/tc-docs/certification/lf-handbook2>
- Exam tips & FAQ: <https://docs.linuxfoundation.org/tc-docs/certification/faq-cka-ckad-cks>
- Kubernetes docs: <https://kubernetes.io/docs/home/>
- `kubectl` cheat sheet: <https://kubernetes.io/docs/reference/kubectl/cheatsheet/>

## Topics

| # | Topic | Domain |
|---|-------|--------|
| 01 | [Kubernetes Architecture](01-Kubernetes-Architecture.md) | Cluster Architecture |
| 02 | [Cluster Installation with kubeadm](02-Cluster-Installation.md) | Cluster Architecture |
| 03 | [kubectl Fundamentals](03-Kubectl-Fundamentals.md) | Cluster Architecture |
| 04 | [Pods](04-Pods.md) | Workloads |
| 05 | [ReplicaSets & ReplicationControllers](05-ReplicaSets.md) | Workloads |
| 06 | [Deployments](06-Deployments.md) | Workloads |
| 07 | [DaemonSets](07-DaemonSets.md) | Workloads |
| 08 | [Jobs & CronJobs](08-Jobs-CronJobs.md) | Workloads |
| 09 | [StatefulSets](09-StatefulSets.md) | Workloads |
| 10 | [Labels, Selectors & Annotations](10-Labels-Selectors-Annotations.md) | Workloads |
| 11 | [Scheduling & Node Affinity](11-Scheduling.md) | Workloads & Scheduling |
| 12 | [Taints & Tolerations](12-Taints-Tolerations.md) | Workloads & Scheduling |
| 13 | [Resource Requests & Limits](13-Resource-Requests-Limits.md) | Workloads & Scheduling |
| 14 | [ConfigMaps & Secrets](14-ConfigMaps-Secrets.md) | Workloads |
| 15 | [Services](15-Services.md) | Services & Networking |
| 16 | [Ingress](16-Ingress.md) | Services & Networking |
| 17 | [Network Policies](17-Network-Policies.md) | Services & Networking |
| 18 | [DNS in Kubernetes (CoreDNS)](18-DNS.md) | Services & Networking |
| 19 | [CNI & Cluster Networking](19-CNI-Networking.md) | Services & Networking |
| 20 | [Gateway API](20-Gateway-API.md) | Services & Networking |
| 21 | [Volumes & Persistent Volumes](21-Volumes-PV-PVC.md) | Storage |
| 22 | [StorageClasses & Dynamic Provisioning](22-StorageClasses.md) | Storage |
| 23 | [Namespaces & Resource Quotas](23-Namespaces-Quotas.md) | Cluster Architecture |
| 24 | [Authentication & Authorization (RBAC)](24-RBAC.md) | Cluster Architecture |
| 25 | [ServiceAccounts](25-ServiceAccounts.md) | Cluster Architecture |
| 26 | [Security Contexts & Admission Control](26-Security-Contexts.md) | Cluster Architecture |
| 27 | [etcd Backup & Restore](27-Etcd-Backup-Restore.md) | Cluster Architecture |
| 28 | [Cluster Upgrades](28-Cluster-Upgrades.md) | Cluster Architecture |
| 29 | [Logging & Monitoring](29-Logging-Monitoring.md) | Troubleshooting |
| 30 | [Troubleshooting Applications](30-Troubleshooting-Applications.md) | Troubleshooting |
| 31 | [Troubleshooting Cluster Components](31-Troubleshooting-Cluster.md) | Troubleshooting |
| 32 | [Helm & Kustomize](32-Helm-Kustomize.md) | Cluster Architecture |
| 33 | [Cheat Sheets & Quick Reference](33-CheatSheets.md) | All |
| 34 | [Multiple Schedulers](34-MultipleSchedulers.md) | Workloads & Scheduling |

---

Good luck — and remember: on the CKA, **speed and accuracy** matter more than memorizing YAML. Learn to generate it.
