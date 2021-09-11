# Installing the Client Tools

In this lab you will install the command line utilities required to complete this tutorial: [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl).

## Install kubectl

The `kubectl` command line utility is used to interact with the Kubernetes API Server. Download and install `kubectl` from the official release binaries:

```sh
wget https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### Verification

Verify `kubectl` version 1.18.6 or higher is installed:
```sh
kubectl version --client
```

> output

```sh
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.6", GitCommit:"dff82dc0de47299ab66c83c626e08b245ab19037", GitTreeState:"clean", BuildDate:"2020-07-15T16:58:53Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}
```

Next: [Provisioning Compute Resources](03-compute-resources.md)
