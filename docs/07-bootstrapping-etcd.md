# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/coreos/etcd).
In this lab you will bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

## Prerequisites

The commands in this lab must be run on each controller instance: `controller-0`, `controller-1`, and `controller-2`.
Login to each controller instance using the `ssh` command. Example:

```sh
for instance in controller-0 controller-1 controller-2; do
  external_ip=$(aws --profile k8s ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  echo ssh -i ~/.ssh/k8s-hard-way.id_rsa ubuntu@$external_ip
done
```

Now ssh into each one of the IP addresses received in last step.

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. See the [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) section in the Prerequisites lab.

## Bootstrapping an etcd Cluster Member

### Download and Install the etcd Binaries

Download the official etcd release binaries from the [coreos/etcd](https://github.com/coreos/etcd) GitHub project:

```sh
wget -q --show-progress --https-only --timestamping "https://github.com/etcd-io/etcd/releases/download/v3.4.10/etcd-v3.4.10-linux-amd64.tar.gz"
```

Extract and install the `etcd` server and the `etcdctl` command line utility:

```sh
{
  tar -xvf etcd-v3.4.10-linux-amd64.tar.gz
  sudo mv etcd-v3.4.10-linux-amd64/etcd* /usr/local/bin/
}
```

### Configure the etcd Server

```sh
{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo chmod 700 /var/lib/etcd
  sudo cp ca.crt kube-apiserver.crt kube-apiserver.key /etc/etcd/
}
```

The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers.
Retrieve the internal IP address for the current compute instance:

```sh
INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
```

Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current compute instance:

```sh
ETCD_NAME=$(curl -s http://169.254.169.254/latest/user-data/ \
  | tr "|" "\n" | grep "^name" | cut -d"=" -f2)
echo "${ETCD_NAME}"
```

Create the `etcd.service` systemd unit file:

```sh
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kube-apiserver.crt \\
  --key-file=/etc/etcd/kube-apiserver.key \\
  --peer-cert-file=/etc/etcd/kube-apiserver.crt \\
  --peer-key-file=/etc/etcd/kube-apiserver.key \\
  --trusted-ca-file=/etc/etcd/ca.crt \\
  --peer-trusted-ca-file=/etc/etcd/ca.crt \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.0.1.10:2380,controller-1=https://10.0.1.11:2380,controller-2=https://10.0.1.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the etcd Server

```sh
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
}
```

> Remember to run the above commands on each controller node: `controller-0`, `controller-1`, and `controller-2`.

## Verification

List the etcd cluster members:

```sh
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/kube-apiserver.crt \
  --key=/etc/etcd/kube-apiserver.key
```

> output

```sh
bbeedf10f5bbaa0c, started, controller-2, https://10.0.1.12:2380, https://10.0.1.12:2379, false
f9b0e395cb8278dc, started, controller-0, https://10.0.1.10:2380, https://10.0.1.10:2379, false
eecdfcb7e79fc5dd, started, controller-1, https://10.0.1.11:2380, https://10.0.1.11:2379, false
```

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
