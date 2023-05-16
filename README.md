# Kubernetes Cluster Mesh with Cilium

Guidelines for setting up a Cluster Mesh with Cilium (tested May 2023).  
Prerequisite: two Kubernetes clusters created with these steps: [kubernetes.md](kubernetes.md). You can skip the deployment of a network plugin part at the end, since for the cluster mesh you end up running more specific `cilium install ...` commands.

## Useful links

 - 📄 [Setting up Cluster Mesh](https://docs.cilium.io/en/v1.13/network/clustermesh/clustermesh/) (official documentation)
 - 🎥 [eCHO Episode 41: Cilium Clustermesh](https://www.youtube.com/watch?v=VBOONHW65NU&t=342s) (Live demo from Liz Rice)
 - 📝 [Deep Dive into Cilium Multi-cluster](https://cilium.io/blog/2019/03/12/clustermesh/) (cilium blogpost from 2019)
 - 📝 [Multi Cluster Networking with Cilium and Friends](https://cilium.io/blog/2022/04/12/cilium-multi-cluster-networking/) (cilium blogpost from 2022)
