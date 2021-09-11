# Provisioning a CA and Generating TLS Certificates

In this lab you will provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure),
then use it to bootstrap a Certificate Authority, and generate TLS certificates for the following components:
* etcd
* kube-apiserver
* kube-controller-manager
* kube-scheduler
* kubelet
* kube-proxy

## Certificate Authority

In this section you will provision a Certificate Authority that can be used to generate additional TLS certificates.

Create a CA certificate, then generate a Certificate Signing Request and use it to create a private key:

```sh
# Create private key for CA
openssl genrsa -out ca.key 2048

# Comment line starting with RANDFILE in /etc/ssl/openssl.cnf definition to avoid permission issues
sudo sed -i '0,/RANDFILE/{s/RANDFILE/\#&/}' /etc/ssl/openssl.cnf

# Create CSR using the private key
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr

# Self sign the csr using its own private key
openssl x509 -req -in ca.csr -signkey ca.key -CAcreateserial  -out ca.crt -days 1000
```
Results:

```sh
ca.crt
ca.key
```

Reference: https://kubernetes.io/docs/concepts/cluster-administration/certificates/#openssl

The `ca.crt` is the Kubernetes Certificate Authority certificate and `ca.key` is the Kubernetes Certificate Authority private key.
You will use the `ca.crt` file in many places, so it will be copied to many places.
The `ca.key` is used by the CA for signing certificates.
And it should be securely stored.

## Client and Server Certificates

In this section you will generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes `admin` user.

### The Admin Client Certificate

Generate the `admin` client certificate and private key:

```sh
# Generate private key for admin user
openssl genrsa -out admin.key 2048

# Generate CSR for admin user. Note the OU.
openssl req -new -key admin.key -subj "/CN=kubernetes-admin/O=system:masters" -out admin.csr

# Sign certificate for admin user using CA servers private key
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out admin.crt -days 1000
```

Note that the admin user is part of the **system:masters** group.
This is how we are able to perform any administrative operations on Kubernetes cluster using kubectl utility.

Results:

```sh
admin.key
admin.crt
```

The `admin.crt` and `admin.key` files give you administrative access.
We will configure these to be used with the kubectl tool to perform administrative functions on kubernetes.

### The Kubelet Client Certificates

```sh
for i in 0 1 2; do
  instance="worker-${i}"
  instance_hostname="ip-10-0-1-2${i}"
  instance_ip="10.0.1.2${i}"
  cat > ${instance}.cnf << EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = ${instance_hostname}

IP.1 = ${instance_ip}
EOF
  openssl genrsa -out ${instance}.key 2048
  openssl req -new -key ${instance}.key -subj "/CN=system:node:${instance_hostname}/O=system:nodes" -out ${instance}.csr -config ${instance}.cnf
  openssl x509 -req -in ${instance}.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out ${instance}.crt -extensions v3_req -extfile ${instance}.cnf -days 1000
done
```

### The Controller Manager Client Certificate

Generate the `kube-controller-manager` client certificate and private key:

```sh
openssl genrsa -out kube-controller-manager.key 2048
openssl req -new -key kube-controller-manager.key -subj "/CN=system:kube-controller-manager" -out kube-controller-manager.csr
openssl x509 -req -in kube-controller-manager.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-controller-manager.crt -days 1000
```

Results:

```sh
kube-controller-manager.key
kube-controller-manager.crt
```

### The Kube Proxy Client Certificate

Generate the `kube-proxy` client certificate and private key:


```sh
openssl genrsa -out kube-proxy.key 2048
openssl req -new -key kube-proxy.key -subj "/CN=system:kube-proxy" -out kube-proxy.csr
openssl x509 -req -in kube-proxy.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-proxy.crt -days 1000
```

Results:

```sh
kube-proxy.key
kube-proxy.crt
```

### The Scheduler Client Certificate

Generate the `kube-scheduler` client certificate and private key:

```sh
openssl genrsa -out kube-scheduler.key 2048
openssl req -new -key kube-scheduler.key -subj "/CN=system:kube-scheduler" -out kube-scheduler.csr
openssl x509 -req -in kube-scheduler.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-scheduler.crt -days 1000
```

Results:

```sh
kube-scheduler.key
kube-scheduler.crt
```

### The Kubernetes API Server Certificate

The kube-apiserver certificate requires all names that various components may reach it to be part of the alternate names.
These include the different DNS names, and IP addresses such as the master servers IP address, the load balancers IP address, the kube-api service IP address etc.

The `openssl` command cannot take alternate names as command line parameter. So we must create a `conf` file for it:

```sh
cat > openssl.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
DNS.5 = ${KUBERNETES_PUBLIC_ADDRESS}
IP.1 = 10.32.0.1
IP.2 = 10.0.1.10
IP.3 = 10.0.1.11
IP.4 = 10.0.1.12
IP.5 = 127.0.0.1
EOF
```

Generates certs for kube-apiserver

```sh
openssl genrsa -out kube-apiserver.key 2048
openssl req -new -key kube-apiserver.key -subj "/CN=kube-apiserver" -out kube-apiserver.csr -config openssl.cnf
openssl x509 -req -in kube-apiserver.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-apiserver.crt -extensions v3_req -extfile openssl.cnf -days 1000
```

Results:

```sh
kube-apiserver.crt
kube-apiserver.key
```

### The ETCD Server Certificate

Similarly ETCD server certificate must have addresses of all the servers part of the ETCD cluster

The `openssl` command cannot take alternate names as command line parameter. So we must create a `conf` file for it:

```sh
cat > openssl-etcd.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
IP.1 = 10.0.1.10
IP.2 = 10.0.1.11
IP.3 = 10.0.1.12
IP.4 = 127.0.0.1
EOF
```

Generates certs for ETCD

```sh
{
  openssl genrsa -out etcd-server.key 2048
  openssl req -new -key etcd-server.key -subj "/CN=etcd-server" -out etcd-server.csr -config openssl-etcd.cnf
  openssl x509 -req -in etcd-server.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out etcd-server.crt -extensions v3_req -extfile openssl-etcd.cnf -days 1000
}
```

Results:

```sh
etcd-server.key
etcd-server.crt
```

## The Service Account Key Pair

The Kubernetes Controller Manager leverages a key pair to generate and sign service account tokens as describe in the [managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/) documentation.

Generate the `service-account` certificate and private key:

```sh
{
  openssl genrsa -out service-account.key 2048
  openssl req -new -key service-account.key -subj "/CN=service-accounts" -out service-account.csr
  openssl x509 -req -in service-account.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out service-account.crt -days 1000
}
```

Results:

```sh
service-account.key
service-account.crt
```

## Distribute the Client and Server Certificates

Copy the appropriate certificates and private keys to each worker instance:

```sh
for instance in worker-0 worker-1 worker-2; do
  external_ip=$(aws --profile k8s ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  scp -i ~/.ssh/k8s-hard-way.id_rsa ca.crt ${instance}.key ${instance}.crt ubuntu@${external_ip}:~/
done
```

Copy the appropriate certificates and private keys to each controller instance:

```sh
for instance in controller-0 controller-1 controller-2; do
  external_ip=$(aws --profile k8s ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  scp -i ~/.ssh/k8s-hard-way.id_rsa \
    ca.crt ca.key kube-apiserver.crt kube-apiserver.key \
    service-account.key service-account.crt ubuntu@${external_ip}:~/
done
```

> The `kube-proxy`, `kube-controller-manager`, `kube-scheduler`, and `kubelet` client certificates will be used to generate client authentication configuration files in the next lab.

Next: [Generating Kubernetes Configuration Files for Authentication](05-kubernetes-configuration-files.md)
