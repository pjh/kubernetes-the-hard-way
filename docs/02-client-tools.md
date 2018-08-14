# Installing the Client Tools

In this lab you will install the command line utilities required to complete this tutorial: [cfssl](https://github.com/cloudflare/cfssl), [cfssljson](https://github.com/cloudflare/cfssl), and [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl).


## Install CFSSL

The `cfssl` and `cfssljson` command line utilities will be used to provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) and generate TLS certificates.

Download and install `cfssl` and `cfssljson` from the [cfssl repository](https://pkg.cfssl.org):

### OS X

```
curl -o cfssl https://pkg.cfssl.org/R1.2/cfssl_darwin-amd64
curl -o cfssljson https://pkg.cfssl.org/R1.2/cfssljson_darwin-amd64
```

```
chmod +x cfssl cfssljson
```

```
sudo mv cfssl cfssljson /usr/local/bin/
```

Some OS X users may experience problems using the pre-built binaries in which case [Homebrew](https://brew.sh) might be a better option:

```
brew install cfssl
```

### Linux

```
wget -q --show-progress --https-only --timestamping \
  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
```

```
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
```

```
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
```

```
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

### Verification

Verify `cfssl` version 1.2.0 or higher is installed:

```
cfssl version
```

> output

```
Version: 1.2.0
Revision: dev
Runtime: go1.6
```

> The cfssljson command line utility does not provide a way to print its version.

## Determine the Kubernetes version

To ensure a consistent Kubernetes version in this guide we'll set a
project-level metadata key. Check the [Kubernetes
Releases](https://github.com/kubernetes/kubernetes/releases) page to find the
latest Kubernetes release or use your preferred version:

```
gcloud compute project-info add-metadata --metadata=k8s-version=v1.11.2
```

## Install kubectl

The `kubectl` command line utility is used to interact with the Kubernetes API
Server. Download and install `kubectl` from the official release binaries:

### OS X

```
K8S_VERSION=`gcloud compute project-info describe --format='value(commonInstanceMetadata.items.k8s-version)'`
curl -o kubectl https://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/darwin/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/local/bin/
```

### Linux

```
{
  K8S_VERSION=`gcloud compute project-info describe --format='value(commonInstanceMetadata.items.k8s-version)'`
  wget https://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kubectl
  chmod +x kubectl
  sudo mv kubectl /usr/local/bin/
}
```

### Verification

Verify `kubectl` version `${K8S_VERSION}` or higher is installed:

```
kubectl version --client
```

> output

```
Client Version: version.Info{Major:"1", Minor:"11", GitVersion:"v1.11.2", GitCommit:"bb9ffb1654d4a729bb4cec18ff088eacc153c239", GitTreeState:"clean", BuildDate:"2018-08-07T23:17:28Z", GoVersion:"go1.10.3", Compiler:"gc", Platform:"linux/amd64"}
```

Next: [Provisioning Compute Resources](03-compute-resources.md)
