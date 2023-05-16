# Kubernetes Cluster Mesh with Cilium

Guidelines for setting up a Cluster Mesh with Cilium (tested May 2023).  
Prerequisite: two Kubernetes clusters created with these steps: [kubernetes.md](kubernetes.md). You can skip the deployment of a network plugin part at the end, since for the cluster mesh you end up running more specific `cilium install ...` commands.

## Kubernetes installation
