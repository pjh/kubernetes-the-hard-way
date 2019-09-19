# Windows Kubernetes The Hard Way

**NOTE:** Windows support in Kubernetes continues to improve and evolve. This
tutorial may still be useful and the resulting cluster may even still work, but
it has not been kept up to date. For the most up-to-date steps for bringing up a
cluster with Windows nodes on Google Compute Engine see
[this code in the main Kubernetes repository](https://github.com/kubernetes/kubernetes/blob/master/cluster/gce/windows/configure.ps1).
For an automated, less-hard way to bring up the cluster check out the
[README](https://github.com/kubernetes/kubernetes/blob/master/cluster/gce/windows/README-GCE-Windows-kube-up.md).

This tutorial walks you through setting up a heterogeneous Kubernetes cluster
that includes both Windows and Linux worker nodes. Kubernetes The Hard Way is
optimized for learning, which means taking the long route to ensure you
understand each task required to bootstrap a Kubernetes cluster.

> The results of this tutorial should not be viewed as production ready, and may receive limited support from the community, but don't let that stop you from learning!

## Target Audience

The target audience for this tutorial is someone trying to run a Windows
Kubernetes cluster for the first time.

## Cluster Details

Kubernetes The Hard Way guides you through bootstrapping a highly available Kubernetes cluster with end-to-end encryption between components and RBAC authentication.

* [Kubernetes](https://github.com/kubernetes/kubernetes) 1.10.2
* [containerd Container Runtime](https://github.com/containerd/containerd) 1.1.0
* [gVisor](https://github.com/google/gvisor) 08879266fef3a67fac1a77f1ea133c3ac75759dd
* [CNI Container Networking](https://github.com/containernetworking/cni) 0.6.0
* [etcd](https://github.com/coreos/etcd) 3.3.5

## Labs

This tutorial assumes you have access to the [Google Cloud Platform](https://cloud.google.com). While GCP is used for basic infrastructure requirements the lessons learned in this tutorial can be applied to other platforms.

* [Glossary](docs/glossary.md)
* [Prerequisites](docs/01-prerequisites.md)
* [Installing the Client Tools](docs/02-client-tools.md)
* [Provisioning Compute Resources](docs/03-compute-resources.md)
* [Provisioning the CA and Generating TLS Certificates](docs/04-certificate-authority.md)
* [Generating Kubernetes Configuration Files for Authentication](docs/05-kubernetes-configuration-files.md)
* [Generating the Data Encryption Config and Key](docs/06-data-encryption-keys.md)
* [Bootstrapping the etcd Cluster](docs/07-bootstrapping-etcd.md)
* [Bootstrapping the Kubernetes Control Plane](docs/08-bootstrapping-kubernetes-controllers.md)
* [Bootstrapping the Linux Worker Nodes](docs/09a-bootstrapping-linux-workers.md)
* [Bootstrapping the Windows Worker Nodes](docs/09b-bootstrapping-windows-workers.md)
* [Configuring kubectl for Remote Access](docs/10-configuring-kubectl.md)
* [Provisioning Pod Network Routes](docs/11-pod-network-routes.md)
* [Deploying the DNS Cluster Add-on](docs/12-dns-addon.md)
* [Smoke Test](docs/13-smoke-test.md)
* [Cleaning Up](docs/14-cleanup.md)
