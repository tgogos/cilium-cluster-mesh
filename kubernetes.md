# install Kubernetes without Docker on Ubuntu 22.04

## VM prerequisites

Make sure the VMs have unique machine-ids and correct hostnames

```bash
sudo rm -rf /etc/machine-id
sudo dbus-uuidgen --ensure=/etc/machine-id
sudo hostnamectl hostname playground-k8s-controller.front.lab
sudo nano /etc/hosts      #add the VM hostname to 127.0.1.1
sudo poweroff

>>> OFFLINE: add Network device      <<<<
>>> OFFLINE: enable QEMU Guest Agent <<<<

sudo apt update
sudo apt install -y qemu-guest-agent
sudo apt dist-upgrade -y
sudo reboot
```

## install `containerd`

After the installation make sure you set `SystemdCgroup` to `true`

```bash
sudo apt install containerd -y
sudo mkdir /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo nano /etc/containerd/config.toml # change to "true" the .runc.options / SystemdCgroup
```


## disable `swap`

```bash
sudo swapoff -a
sudo sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab
```



## configure some networking "gotchas"

```bash
sudo nano /etc/sysctl.conf               # uncomment net.ipv4.ip_forward=1
sudo touch /etc/modules-load.d/k8s.conf
echo "br_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
sudo reboot
```




## install the Kubernetes-related software

```bash
# old: sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
# old: echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubeadm kubectl kubelet
```




## Cluster initialization & CNI plugin

```bash
sudo kubeadm init --control-plane-endpoint=10.220.2.147 --node-name playground-k8s-controller.front.lab --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get pods --all-namespaces
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
kubectl get pods --all-namespaces
```

For `cilium` instead of `Flannel`:

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

cilium install
```
