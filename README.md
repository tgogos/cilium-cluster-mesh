# Kubernetes Cluster Mesh with Cilium

Guidelines for setting up a Cluster Mesh with Cilium (tested May 2023).  
Prerequisite: two Kubernetes clusters created with these steps: [kubernetes.md](kubernetes.md). You can skip the deployment of a network plugin part at the end, since for the cluster mesh you end up running more specific `cilium install ...` commands.

## Useful links

 - 📄 [Setting up Cluster Mesh](https://docs.cilium.io/en/v1.13/network/clustermesh/clustermesh/) (cilium official documentation)
 - 🎥 [eCHO Episode 41: Cilium Clustermesh](https://www.youtube.com/watch?v=VBOONHW65NU&t=342s) (Live demo from Liz Rice)
 - 📝 [Deep Dive into Cilium Multi-cluster](https://cilium.io/blog/2019/03/12/clustermesh/) (cilium blogpost from 2019)
 - 📝 [Multi Cluster Networking with Cilium and Friends](https://cilium.io/blog/2022/04/12/cilium-multi-cluster-networking/) (cilium blogpost from 2022)
 - 📄 [Load-balancing & Service Discovery](https://docs.cilium.io/en/v1.13/network/clustermesh/services/) (cilium official documentation)


## Setup

ℹ️ *The following steps use the `--context` option both for `kubectl ...` and `cilium ...` commands. They were run from a VM that was not part of any of the clusters. All the VMs had network connectivity within a Proxmox cluster, no VPN was used.*

```
           ┌──────────────────────────┐
           │                          │
           │   VM with kubectl that   │
           │  accesses both clusters  │
           │                          │
           └──────┬───────────┬───────┘
                  │           │
                  │           │
┌─────────────────┴──┐     ┌──┴──────────────────┐
│                    │     │                     │
│      cluster 1     │     │      cluster 2      │
│                    │     │                     │
└────────────────────┘     └─────────────────────┘
```

### cluster mesh creation - `cilium` commands

```
# uninstall step / used because the clusters were already using cilium as the CNI plugin)
cilium uninstall --context $CLUSTER1
cilium uninstall --context $CLUSTER2
```

```
# installation step
cilium install --context $CLUSTER1 --cluster-name kubernetes-1 --cluster-id 1 --ipv4-native-routing-cidr=10.0.0.0/9
cilium install --context $CLUSTER2 --cluster-name kubernetes-2 --cluster-id 2 --ipv4-native-routing-cidr=10.0.0.0/9
```

```
# enable Cluster Mesh step
cilium clustermesh enable --context $CLUSTER1 --service-type NodePort
cilium clustermesh enable --context $CLUSTER2 --service-type NodePort

cilium clustermesh status --context $CLUSTER1 --wait
cilium clustermesh status --context $CLUSTER2 --wait
```

```
# connect clusters step
cilium clustermesh connect --context $CLUSTER1 --destination-context $CLUSTER2
```

![](/imgs/clustermesh-connect_and_status.png)

### test pod connectivity

```
cilium connectivity test --context $CLUSTER1 --multi-cluster $CLUSTER2
```

<details>
<summary>result:</summary>

```
$ cilium connectivity test --context $CLUSTER1 --multi-cluster $CLUSTER2
ℹ️  Monitor aggregation detected, will skip some flow validation steps
✨ [kubernetes-1] Creating namespace cilium-test for connectivity check...
✨ [kubernetes-2] Creating namespace cilium-test for connectivity check...
✨ [kubernetes-1] Deploying echo-same-node service...
✨ [kubernetes-1] Deploying echo-other-node service...
✨ [kubernetes-1] Deploying DNS test server configmap...
✨ [kubernetes-2] Deploying DNS test server configmap...
✨ [kubernetes-1] Deploying same-node deployment...
✨ [kubernetes-1] Deploying client deployment...
✨ [kubernetes-1] Deploying client2 deployment...
✨ [kubernetes-2] Deploying echo-other-node service...
✨ [kubernetes-2] Deploying other-node deployment...
⌛ [kubernetes-1] Waiting for deployments [client client2 echo-same-node] to become ready...
⌛ [kubernetes-2] Waiting for deployments [echo-other-node] to become ready...
⌛ [kubernetes-1] Waiting for CiliumEndpoint for pod cilium-test/client-6965d549d5-7vhpz to appear...
⌛ [kubernetes-1] Waiting for CiliumEndpoint for pod cilium-test/client2-76f4d7c5bc-bx6kz to appear...
⌛ [kubernetes-1] Waiting for pod cilium-test/client2-76f4d7c5bc-bx6kz to reach DNS server on cilium-test/echo-same-node-799c9b99f-gwb55 pod...
⌛ [kubernetes-1] Waiting for pod cilium-test/client-6965d549d5-7vhpz to reach DNS server on cilium-test/echo-same-node-799c9b99f-gwb55 pod...
⌛ [kubernetes-1] Waiting for pod cilium-test/client-6965d549d5-7vhpz to reach DNS server on cilium-test/echo-other-node-f57db5457-njphn pod...
⌛ [kubernetes-1] Waiting for pod cilium-test/client2-76f4d7c5bc-bx6kz to reach DNS server on cilium-test/echo-other-node-f57db5457-njphn pod...
⌛ [kubernetes-1] Waiting for pod cilium-test/client2-76f4d7c5bc-bx6kz to reach default/kubernetes service...
⌛ [kubernetes-1] Waiting for pod cilium-test/client-6965d549d5-7vhpz to reach default/kubernetes service...
⌛ [kubernetes-1] Waiting for CiliumEndpoint for pod cilium-test/echo-same-node-799c9b99f-gwb55 to appear...
⌛ [kubernetes-2] Waiting for CiliumEndpoint for pod cilium-test/echo-other-node-f57db5457-njphn to appear...
⌛ [kubernetes-1] Waiting for Service cilium-test/echo-other-node to become ready...
⌛ [kubernetes-1] Waiting for Service cilium-test/echo-same-node to become ready...
ℹ️  Skipping IPCache check
🔭 Enabling Hubble telescope...
⚠️  Unable to contact Hubble Relay, disabling Hubble telescope and flow validation: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp 127.0.0.1:4245: connect: connection refused"
ℹ️  Expose Relay locally with:
   cilium hubble enable
   cilium hubble port-forward&
ℹ️  Cilium version: 1.13.2
🏃 Running tests...
[=] Test [no-policies]
..........................

[=] Skipping Test [no-policies-extra]
[=] Test [allow-all-except-world]
................
[=] Test [client-ingress]
..
[=] Test [client-ingress-knp]
..
[=] Test [all-ingress-deny]
........
[=] Test [all-ingress-deny-knp]
........
[=] Test [all-egress-deny]
................
[=] Test [all-egress-deny-knp]
................
[=] Test [all-entities-deny]
........
[=] Test [cluster-entity]
..
[=] Test [cluster-entity-multi-cluster]
..
[=] Test [host-entity]
......
[=] Test [echo-ingress]
....
[=] Test [echo-ingress-knp]
....
[=] Test [client-ingress-icmp]
..
[=] Test [client-egress]
....
[=] Test [client-egress-knp]
....
[=] Test [client-egress-expression]
....
[=] Test [client-egress-expression-knp]
....
[=] Test [client-with-service-account-egress-to-echo]
....
[=] Test [client-egress-to-echo-service-account]
....
[=] Test [to-entities-world]
......
[=] Test [to-cidr-external]
....
[=] Test [to-cidr-external-knp]
....
[=] Test [echo-ingress-from-other-client-deny]
......
[=] Test [client-ingress-from-other-client-icmp-deny]
......
[=] Test [client-egress-to-echo-deny]
......
[=] Test [client-ingress-to-echo-named-port-deny]
....
[=] Test [client-egress-to-echo-expression-deny]
....
[=] Test [client-with-service-account-egress-to-echo-deny]
....
[=] Test [client-egress-to-echo-service-account-deny]
..
[=] Test [client-egress-to-cidr-deny]
....
[=] Test [client-egress-to-cidr-deny-default]
....
[=] Test [health]
.....
[=] Test [echo-ingress-l7]
............
[=] Test [echo-ingress-l7-named-port]
............
[=] Test [client-egress-l7-method]
............
[=] Test [client-egress-l7]
..........
[=] Test [client-egress-l7-named-port]
..........

[=] Skipping Test [client-egress-l7-tls-deny-without-headers]

[=] Skipping Test [client-egress-l7-tls-headers]

[=] Skipping Test [client-egress-l7-set-header]

[=] Skipping Test [echo-ingress-auth-always-fail]

[=] Skipping Test [echo-ingress-auth-mtls-spiffe]

[=] Skipping Test [pod-to-ingress-service]

[=] Skipping Test [pod-to-ingress-service-deny-all]

[=] Skipping Test [pod-to-ingress-service-allow-ingress-identity]
[=] Test [dns-only]
..........
[=] Test [to-fqdns]
........

✅ All 41 tests (279 actions) successful, 9 tests skipped, 1 scenarios skipped.
```
 
 </details>

<br><br><br>





## Add a 3rd cluster to the mix!

The steps below include details on how to add a 3rd cluster which isn't connected to the same network with the first two. Our setup had `kubernetes-1` and `kubernetes-2` in the same lab environment (with network connectivity provided) and now `kubernetes-3` lives inside a DigitalOcean VM and connects to our lab with `openvpn`.

    openvpn3 session-start --config <your_.ovpn_file_here>

Cluster initialization:

    kubeadm init --config kubeadm-init-configuration.yaml --ignore-preflight-errors=NumCPU

Make master node capable of scheduling pods:

    kubectl taint node <your_node_name> node-role.kubernetes.io/control-plane:NoSchedule-

Cilium clustermesh setup:

    cilium uninstall --context $CLUSTER3
    cilium install --context $CLUSTER3 --cluster-name kubernetes-3 --cluster-id 3 --ipv4-native-routing-cidr=10.0.0.0/9
    cilium clustermesh enable --context $CLUSTER3 --service-type NodePort
    cilium clustermesh status --context $CLUSTER3 --wait
    
    # connect the clusters 1-3 & 2-3
    cilium clustermesh connect --context $CLUSTER1 --destination-context $CLUSTER3
    cilium clustermesh connect --context $CLUSTER2 --destination-context $CLUSTER3


![](/imgs/clustermesh-3-clusters-connected.png)

<br><br><br>




## Load-balancing & Service Discovery

The instructions come directly from the docs.

    kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.13/examples/kubernetes/clustermesh/global-service-example/cluster1.yaml
    kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.13/examples/kubernetes/clustermesh/global-service-example/cluster2.yaml

For the `kubernetes-3` deployment, download the yaml file and modify accordingly so that the service replies with a `...Cluster-3` message.

Load-balancing in action (from inside a pod):

```bash
$ kubectl exec -ti deployment/x-wing -- bash
root@x-wing-64665f7b7b-ksmdk:/curl rebel-base
{"Galaxy": "Alderaan", "Cluster": "Cluster-1"}
root@x-wing-64665f7b7b-ksmdk:/# curl rebel-base
{"Galaxy": "Alderaan", "Cluster": "Cluster-2"}
root@x-wing-64665f7b7b-ksmdk:/# curl rebel-base
{"Galaxy": "Alderaan", "Cluster": "Cluster-1"}
root@x-wing-64665f7b7b-ksmdk:/# curl rebel-base
{"Galaxy": "Alderaan", "Cluster": "Cluster-3"}
root@x-wing-64665f7b7b-ksmdk:/# curl rebel-base
{"Galaxy": "Alderaan", "Cluster": "Cluster-3"}
root@x-wing-64665f7b7b-ksmdk:/# curl rebel-base
{"Galaxy": "Alderaan", "Cluster": "Cluster-2"}
root@x-wing-64665f7b7b-ksmdk:/# curl rebel-base
{"Galaxy": "Alderaan", "Cluster": "Cluster-3"}
root@x-wing-64665f7b7b-ksmdk:/# curl rebel-base
{"Galaxy": "Alderaan", "Cluster": "Cluster-1"}
root@x-wing-64665f7b7b-ksmdk:/# curl rebel-base
{"Galaxy": "Alderaan", "Cluster": "Cluster-3"}
root@x-wing-64665f7b7b-ksmdk:/# curl rebel-base
{"Galaxy": "Alderaan", "Cluster": "Cluster-1"}
root@x-wing-64665f7b7b-ksmdk:/# curl rebel-base
{"Galaxy": "Alderaan", "Cluster": "Cluster-2"}
root@x-wing-64665f7b7b-ksmdk:/# curl rebel-base
{"Galaxy": "Alderaan", "Cluster": "Cluster-1"}
root@x-wing-64665f7b7b-ksmdk:/# curl rebel-base
{"Galaxy": "Alderaan", "Cluster": "Cluster-1"}
root@x-wing-64665f7b7b-ksmdk:/# curl rebel-base
{"Galaxy": "Alderaan", "Cluster": "Cluster-3"}
root@x-wing-64665f7b7b-ksmdk:/# curl rebel-base
{"Galaxy": "Alderaan", "Cluster": "Cluster-2"}
root@x-wing-64665f7b7b-ksmdk:/# curl rebel-base
{"Galaxy": "Alderaan", "Cluster": "Cluster-1"}
```
