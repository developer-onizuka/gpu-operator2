# GPU Operator with preinstalled driver in Host (the Case #3 below)

-One master node (No GPU machine)

-Two worker nodes (GPU machine and CPU machine)

https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/getting-started.html#chart-customization-options

| # | Scenario | Nvidia Driver | Nvidia Toolkit |
| --- | --- | --- | --- |
| #1 | No GPU operator | In Host | In Host |
| #2 | GPU Operator (default) | DaemonSet | DaemonSet |
| #3 | GPU Operator (w/ driver.enabled=false) | In Host | DaemonSet |
| #4 | GPU Operator (w/ toolkit.enabled=false) | DaemonSet | In Host |

# 1. Master node (no GPU machine)
# 1-1. Disable Swapping and Blacklisting Nouveau driver
```
$ uname -a
Linux worker 5.11.0-27-generic #29~20.04.1-Ubuntu SMP Wed Aug 11 15:58:17 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
$ sudo vi /etc/fstab
# comment out like below:
#/swapfile                                 none            swap    sw              0       0
-----
$ sudo swapoff -a
$ sudo systemctl mask "swapfile.swap"
Created symlink /etc/systemd/system/swapfile.swap → /dev/null.
```
The nouveau driver for NVIDIA GPUs must be blacklisted before starting the GPU Operator. 
Create a file at /etc/modprobe.d/blacklist-nouveau.conf with the following contents:
```
$ sudo vi /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0

$ sudo update-initramfs -u
```
# 1-2. Install Curl
```
$ sudo apt-get install curl
```
# 1-3. Install Docker-CE
```
$ curl https://get.docker.com | sh \
&& sudo systemctl --now enable docker
```
# 1-4. Configuring about using systemd instead of cgroups.
```
$ cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

$ sudo systemctl enable docker
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```
# 1-5. Install kubernetes
```
$ sudo apt-get update \
&& sudo apt-get install -y apt-transport-https curl

$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

$ sudo apt-get update \
&& sudo apt-get install -y -q kubelet kubectl kubeadm \
&& sudo kubeadm init --pod-network-cidr=192.168.0.0/16

Copy the output below:
********************************************************************************************************
kubeadm join 192.168.122.147:6443 --token xxxxxxxxxxxxxxxxxxxxxxx \
	--discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
********************************************************************************************************

$ mkdir -p $HOME/.kube \
&& sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config \
&& sudo chown $(id -u):$(id -g) $HOME/.kube/config

$ kubectl taint nodes --all node-role.kubernetes.io/master-
node/master untainted

$ kubectl get nodes
NAME     STATUS     ROLES                  AGE     VERSION
master   NotReady   control-plane,master   8m41s   v1.22.1
```



