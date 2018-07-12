# Glossary

TODO: add a glossary of terms here and their corresponding values and how to get
them. E.g.: service cluster CIDR, service cluster DNS IP, pod CIDR, etc. etc.

## Networking

VPC subnet: the subnet of the GCE virtual private cloud network from which
internal IP addresses for each VM instance will be taken.
  - The VPC subnet has no relation to the cluster CIDR and must not overlap
    with the cluster CIDR.
  - 10.240.0.0/24 in this tutorial.

Cluster CIDR: a subnet in private IP address space, from which IP addresses for
all Kubernetes components will be taken. The cluster CIDR is carved into smaller
pod CIDRs.
  - The cluster CIDR has no relation to the VPC subnet and must not overlap with
    it.
  - 10.200.0.0/16 in this tutorial.

Pod CIDR: a subnet of the cluster CIDR that is assigned to a specific worker
node, from which pod IP addresses will be taken.
  - This tutorial uses instance metadata as the mechanism for telling each
    worker node its pod CIDR. To fetch the pod CIDR from within an instance,
    run:
      - Linux: TODO
      - Windows: TODO
  - For worker node N in this tutorial, the pod CIDR is 10.200.N.0/24

## Kubernetes components

Admin: (TODO: is this a real component?)

API Server: TODO

Controller Manager: TODO

Kube Proxy: TODO

Kube Scheduler: TODO

Service Account: TODO


