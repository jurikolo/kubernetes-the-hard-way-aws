# Bootstrapping the Kubernetes Worker Nodes

In this lab you will bootstrap three Kubernetes worker nodes.
The following components will be installed on each node:
* [runc](https://github.com/opencontainers/runc)
* [gVisor](https://github.com/google/gvisor)
* [container networking plugins](https://github.com/containernetworking/cni)
* [containerd](https://github.com/containerd/containerd)
* [kubelet](https://kubernetes.io/docs/admin/kubelet)
* [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

## Prerequisites

The commands in this lab must be run on each worker instance: `worker-0`, `worker-1`, and `worker-2`.
Login to each worker instance using the `ssh` command. Example:

```sh
for instance in worker-0 worker-1 worker-2; do
  external_ip=$(aws --profile k8s ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  echo ssh -i ~/.ssh/k8s-hard-way.id_rsa ubuntu@$external_ip
done
```

Now ssh into each one of the IP addresses received in last step.

## Provisioning a Kubernetes Worker Node

Install the OS dependencies:

```sh
{
  sudo apt-get update
  sudo apt-get -y install socat conntrack ipset
}
```

> The socat binary enables support for the `kubectl port-forward` command.

### Download and Install Worker Binaries

```sh
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.18.0/crictl-v1.18.0-linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-the-hard-way/runsc \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc91/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-amd64-v0.8.6.tgz \
  https://github.com/containerd/containerd/releases/download/v1.3.6/containerd-1.3.6-linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kubelet
```

Create the installation directories:

```sh
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

Install the worker binaries:

```sh
{
  chmod +x kubectl kube-proxy kubelet runc.amd64 runsc
  sudo mv runc.amd64 runc
  sudo mv kubectl kube-proxy kubelet runc runsc /usr/local/bin/
  sudo tar -xvf crictl-v1.18.0-linux-amd64.tar.gz -C /usr/local/bin/
  sudo tar -xvf cni-plugins-linux-amd64-v0.8.6.tgz -C /opt/cni/bin/
  sudo tar -xvf containerd-1.3.6-linux-amd64.tar.gz
  sudo mv bin/* /bin/
}
```

### Configure CNI Networking

Retrieve the Pod CIDR range for the current compute instance:

```sh
POD_CIDR=$(curl -s http://169.254.169.254/latest/user-data/ \
  | tr "|" "\n" | grep "^pod-cidr" | cut -d"=" -f2)
echo "${POD_CIDR}"
```

Create the `bridge` network configuration file:

```sh
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

Create the `loopback` network configuration file:

```sh
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.3.1",
    "name": "lo",
    "type": "loopback"
}
EOF
```

### Configure containerd

Create the `containerd` configuration file:

```sh
sudo mkdir -p /etc/containerd/
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
    [plugins.cri.containerd.untrusted_workload_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runsc"
      runtime_root = "/run/containerd/runsc"
EOF
```

> Untrusted workloads will be run using the gVisor (runsc) runtime.

Create the `containerd.service` systemd unit file:

```sh
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubelet

```sh
{
  WORKER_NAME=$(curl -s http://169.254.169.254/latest/user-data/ | tr "|" "\n" | grep "^name" | cut -d"=" -f2)
  echo "${WORKER_NAME}"

  sudo mv ${WORKER_NAME}.key ${WORKER_NAME}.crt /var/lib/kubelet/
  sudo mv ${WORKER_NAME}.kubeconfig /var/lib/kubelet/kubeconfig
  sudo mv ca.crt /var/lib/kubernetes/
}
```

Create the `kubelet-config.yaml` configuration file:

```sh
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.crt"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${WORKER_NAME}.crt"
tlsPrivateKeyFile: "/var/lib/kubelet/${WORKER_NAME}.key"
resolvConf: "/run/systemd/resolve/resolv.conf"
EOF
```

Create the `kubelet.service` systemd unit file:

```sh
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Proxy

```sh
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

Create the `kube-proxy-config.yaml` configuration file:

```sh
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

Create the `kube-proxy.service` systemd unit file:

```sh
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the Worker Services

```sh
{
  sudo systemctl daemon-reload
  sudo systemctl enable containerd kubelet kube-proxy
  sudo systemctl start containerd kubelet kube-proxy
}
```

## Verification

> The compute instances created in this tutorial will not have permission to complete this section. Run the following commands from the same machine used to create the compute instances.

List the registered Kubernetes nodes:

```sh
external_ip=$(aws --profile k8s ec2 describe-instances --filters \
    "Name=tag:Name,Values=controller-0" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

ssh -i k8s-hard-way.id_rsa ubuntu@${external_ip}

kubectl get nodes --kubeconfig admin.kubeconfig
```

> output

```
NAME             STATUS   ROLES    AGE   VERSION
ip-10-0-1-20   Ready    <none>   51s   v1.18.6
ip-10-0-1-21   Ready    <none>   51s   v1.18.6
ip-10-0-1-22   Ready    <none>   51s   v1.18.6
```

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
