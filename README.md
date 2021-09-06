# GPU Operator with preinstalled driver in Host (the Case #2 below)

-One master node (No GPU machine)

-Two worker nodes (GPU machine and CPU machine)

https://youtu.be/NrjNibxd1rk

|  | CPU | Memory | GPU | GPU Driver |
| --- | --- | --- | --- | --- |
| Master | 2 | 8,192 MB | no | --- |
| Worker1 | 1 | 8,192 MB | 1 | Installed |
| Worker2 | 2 | 8,192 MB | no | --- |

https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/getting-started.html#chart-customization-options

| # | Scenario | Nvidia Driver | Nvidia Toolkit |
| --- | --- | --- | --- |
| #1 | No GPU operator | In Host | In Host |
| #2 | GPU Operator (w/ driver.enabled=false) | In Host | DaemonSet |
| #3 | GPU Operator (default) | DaemonSet | DaemonSet |
| #4 | GPU Operator (w/ toolkit.enabled=false) | DaemonSet | In Host |

# 1. Master node
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

# 2. Worker node1 (GPU machine) and Worker node2 (CPU macine)
# 2-1. Disable Swapping and Blacklisting Nouveau driver
Same as #1-1.

# 2-2. Install Nvidia-driver (Only GPU machine)
```
$ sudo apt-get update
$ sudo apt-get install nvidia-driver-470
$ reboot
```

# 2-3. Install Curl
```
$ sudo apt-get install curl
```
# 2-4. Install Docker-CE
```
$ curl https://get.docker.com | sh \
&& sudo systemctl --now enable docker
```
# 2-5. Configuring about using systemd instead of cgroups.
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
# 2-6. Install kubernetes
```
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

$ sudo apt-get update \
&& sudo apt-get install -y -q kubelet kubectl kubeadm
```
# 2-7. Join cluster
```
$ sudo kubeadm join 192.168.122.147:6443 --token xxxxxxxxxxxxxxxxxxxxxxx \
	--discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

If you forget it, you can confirm it again as below:
```
$ kubeadm token list
$ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
>    openssl dgst -sha256 -hex | sed 's/^.* //'
```

If the token was expired (no result of "kubeadm token list"), then you can create it again.
```
$ kubeadm token create
```

# 3. Master node
# 3-1. Comfirm the nodes in the cluster at Master node
```
$ kubectl get nodes
NAME      STATUS     ROLES                  AGE   VERSION
master    NotReady   control-plane,master   23m   v1.22.1
worker1   NotReady   <none>                 32s   v1.22.1
worker2   NotReady   <none>                 20s   v1.22.1
```
# 3-2. Label node at Master node
```
$ kubectl label node worker1 node-role.kubernetes.io/node=worker1
node/worker1 labeled
$ kubectl label node worker2 node-role.kubernetes.io/node=worker2
node/worker2 labeled

$ kubectl get nodes
NAME      STATUS     ROLES                  AGE     VERSION
master    NotReady   control-plane,master   26m     v1.22.1
worker1   NotReady   node                   9m6s    v1.22.1
worker2   NotReady   node                   3m47s   v1.22.1
```

# 3-3. Install contoller Pods at Master node 
```
$ kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Images in Master node
```
master:~/Desktop$ sudo docker images
REPOSITORY                           TAG       IMAGE ID       CREATED        SIZE
k8s.gcr.io/kube-apiserver            v1.22.1   f30469a2491a   2 weeks ago    128MB
k8s.gcr.io/kube-controller-manager   v1.22.1   6e002eb89a88   2 weeks ago    122MB
k8s.gcr.io/kube-scheduler            v1.22.1   aca5ededae9c   2 weeks ago    52.7MB
k8s.gcr.io/kube-proxy                v1.22.1   36c4ebbc9d97   2 weeks ago    104MB
calico/node                          v3.20.0   5ef66b403f4f   5 weeks ago    170MB
calico/pod2daemon-flexvol            v3.20.0   5991877ebc11   5 weeks ago    21.7MB
calico/cni                           v3.20.0   4945b742b8e6   5 weeks ago    146MB
k8s.gcr.io/etcd                      3.5.0-0   004811815584   2 months ago   295MB
k8s.gcr.io/coredns/coredns           v1.8.4    8d147537fb7d   3 months ago   47.6MB
k8s.gcr.io/pause                     3.5       ed210e3e4a5b   5 months ago   683kB
```

Images in Worker1 and Worker2 node
```
worker1:~/Desktop$ sudo docker images
REPOSITORY                  TAG       IMAGE ID       CREATED        SIZE
k8s.gcr.io/kube-proxy       v1.22.1   36c4ebbc9d97   2 weeks ago    104MB
calico/node                 v3.20.0   5ef66b403f4f   5 weeks ago    170MB
calico/pod2daemon-flexvol   v3.20.0   5991877ebc11   5 weeks ago    21.7MB
calico/cni                  v3.20.0   4945b742b8e6   5 weeks ago    146MB
k8s.gcr.io/pause            3.5       ed210e3e4a5b   5 months ago   683kB
```

# 4. Master node
# 4-1. Install Helm chart at Master node
```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 \
&& chmod 700 get_helm.sh \
&& ./get_helm.sh

$ helm repo add nvidia https://nvidia.github.io/gpu-operator \
&& helm repo update

$ helm install --wait --generate-name \
nvidia/gpu-operator \
--set driver.enabled=false

$ kubectl get pods -A
```

# 4-2. Run yaml file without GPU at Master node
```
$ cat ubuntu.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu 
  labels:
    name: ubuntu
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    command:
    - sleep
    - "3600"
 
$ kubectl apply -f ubuntu.yaml
pod/ubuntu created

$ kubectl get pods
NAME                                                              READY   STATUS    RESTARTS      AGE
gpu-operator-1630767025-node-feature-discovery-master-7f8bshkmt   1/1     Running   1 (8h ago)    8h
gpu-operator-1630767025-node-feature-discovery-worker-2npzh       1/1     Running   1 (8h ago)    8h
gpu-operator-1630767025-node-feature-discovery-worker-2z84j       1/1     Running   0             9m21s
gpu-operator-1630767025-node-feature-discovery-worker-rjbf4       1/1     Running   2 (24m ago)   8h
gpu-operator-74dcf6544d-7xg48                                     1/1     Running   1 (8h ago)    8h
ubuntu                                                            1/1     Running   0             29s

$ kubectl describe pod ubuntu
Name:         ubuntu
Namespace:    default
Priority:     0
Node:         worker2/192.168.122.77
Start Time:   Sun, 05 Sep 2021 08:31:04 +0900
Labels:       name=ubuntu
Annotations:  cni.projectcalico.org/containerID: 0605982c33258e76647736a32a1f66b54811ab1fc5dcbbadda09f3dbcc0286a3
              cni.projectcalico.org/podIP: 192.168.189.66/32
              cni.projectcalico.org/podIPs: 192.168.189.66/32
Status:       Running
IP:           192.168.189.66
IPs:
  IP:  192.168.189.66
Containers:
  ubuntu:
    Container ID:  docker://f4fe7eb6ac0faaad77eaa2d4c06b9945568ffe35c6e369401b6a4a0e87d6f07d
    Image:         ubuntu
    Image ID:      docker-pullable://ubuntu@sha256:9d6a8699fb5c9c39cf08a0871bd6219f0400981c570894cd8cbea30d3424a31f
    Port:          <none>
    Host Port:     <none>
    Command:
      sleep
      3600
    State:          Running
      Started:      Sun, 05 Sep 2021 08:31:20 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-mbtwg (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-mbtwg:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  60s   default-scheduler  Successfully assigned default/ubuntu to worker2
  Normal  Pulling    59s   kubelet            Pulling image "ubuntu"
  Normal  Pulled     44s   kubelet            Successfully pulled image "ubuntu" in 14.789718728s
  Normal  Created    44s   kubelet            Created container ubuntu
  Normal  Started    44s   kubelet            Started container ubuntu


$ kubectl exec -it ubuntu -- /bin/bash
root@ubuntu:/# 
root@ubuntu:/# nvidia-smi

$ kubectl delete pod ubuntu
```

# 4-3. Run yaml file with GPU at Master node
```
$ cat ubuntu-gpu.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu 
  labels:
    name: ubuntu
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    command:
    - sleep
    - "3600"
    resources:
      limits:
         nvidia.com/gpu: 1

$ kubectl apply -f ubuntu-gpu.yaml
pod/ubuntu created

$ kubectl get pods
NAME                                                              READY   STATUS    RESTARTS   AGE
gpu-operator-1630767025-node-feature-discovery-master-7f8bshkmt   1/1     Running   0          12m
gpu-operator-1630767025-node-feature-discovery-worker-2npzh       1/1     Running   0          12m
gpu-operator-1630767025-node-feature-discovery-worker-rjbf4       1/1     Running   0          12m
gpu-operator-74dcf6544d-7xg48                                     1/1     Running   0          12m
ubuntu                                                            1/1     Running   0          75s

$ kubectl describe pod ubuntu
Name:         ubuntu
Namespace:    default
Priority:     0
Node:         worker1/192.168.122.202
Start Time:   Sun, 05 Sep 2021 00:01:34 +0900
Labels:       name=ubuntu
Annotations:  cni.projectcalico.org/containerID: eeb293bd1559f8a08a67dbcdcd57dfb3e224d531a90216bb324c13a9b635d575
              cni.projectcalico.org/podIP: 192.168.235.145/32
              cni.projectcalico.org/podIPs: 192.168.235.145/32
Status:       Running
IP:           192.168.235.145
IPs:
  IP:  192.168.235.145
Containers:
  ubuntu:
    Container ID:  docker://00e42de0040646c95104fba8082ebc297ca3ac13e891f74e6c64b48897e231b1
    Image:         ubuntu
    Image ID:      docker-pullable://ubuntu@sha256:9d6a8699fb5c9c39cf08a0871bd6219f0400981c570894cd8cbea30d3424a31f
    Port:          <none>
    Host Port:     <none>
    Command:
      sleep
      3600
    State:          Running
      Started:      Sun, 05 Sep 2021 00:01:57 +0900
    Ready:          True
    Restart Count:  0
    Limits:
      nvidia.com/gpu:  1
    Requests:
      nvidia.com/gpu:  1
    Environment:       <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-7nf2f (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-7nf2f:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  3m15s  default-scheduler  Successfully assigned default/ubuntu to worker1
  Normal  Pulling    3m14s  kubelet            Pulling image "ubuntu"
  Normal  Pulled     2m53s  kubelet            Successfully pulled image "ubuntu" in 21.527365695s
  Normal  Created    2m52s  kubelet            Created container ubuntu
  Normal  Started    2m52s  kubelet            Started container ubuntu

$ kubectl exec -it ubuntu -- /bin/bash
root@ubuntu:/# 
root@ubuntu:/# nvidia-smi
Sat Sep  4 15:03:16 2021       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 470.57.02    Driver Version: 470.57.02    CUDA Version: 11.4     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Quadro P1000        Off  | 00000000:04:00.0 Off |                  N/A |
| 34%   38C    P8    N/A /  N/A |     11MiB /  4040MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
+-----------------------------------------------------------------------------+

```

# 5. Uninstall GPU operator
```
$ helm delete $(helm ls -n default | awk '/gpu-operator/{print $1}') -n default
```

# 6. Delete worker nodes
```
$ kubectl delete node worker1
$ kubectl delete node worker2
```
