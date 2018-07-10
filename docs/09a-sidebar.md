"Management" CIDR / IP below: these are the address(es) associated with the
vEthernet interface(s) on the Windows node (see AddRoutes.ps1 script). The
Microsoft scripts (AddRoutes.ps1 and start-kubelet.ps1) assume that the node has
a "vEthernet (Ethernet*" interface as a prerequisite. However, my Windows nodes
only have "vEthernet (nat)" to start. wtf?

I think this is because my Docker network is not configured or created properly.
For OVN / OVS docker is started with "bridge" : "none" by configuring
C:\ProgramData\docker\config\daemon.json
(https://github.com/apprenda/kubernetes-ovn-heterogeneous-cluster/search?q=programdata&unscoped_q=programdata);
is this "transparent" network mode? It seems like what I want is "l2bridge"
network mode: how do I get that? Do I need to update the daemon.json file, or do
I need to do a `docker network create` command?
(https://docs.microsoft.com/en-us/virtualization/windowscontainers/container-networking/network-drivers-topologies).

I finally found
https://github.com/Microsoft/SDN/blob/master/Containers/README.md#create-an-l2bridge-container-network
which suggests that I can or must do a docker network create for an l2bridge
network. However, what subnet do I use?? That sample command uses the
192.168.1.0/24 network which is the *pod CIDR*, so I guess I'll do that too
here. Should the gateway be 10.200.1.1 or 10.200.1.2? Who the hell knows, but probably .2.

A-HA: I found what I was missing: in start-kubelet.ps1 there's a call to
New-HNSNetwork which creates the l2bridge network. The -AddressPrefix is the
$podCIDR and the -Gateway is the $podGW which comes from Get-PodGateway which
uses the .1 address, not the .2 address!

Is it possible to replace this scripts dumb New-HNSNetwork call with a simple
docker network create command? Unclear, but seems plausible. So, my initial
approach: for worker-1:

```
docker network create -d l2bridge --subnet="10.200.1.0/24" --gateway="10.200.1.1" l2bridge
# RDP connection may fail or hiccup

docker network ls
```

> NETWORK ID          NAME                DRIVER              SCOPE
> fe56e4376d30        l2bridge            l2bridge            local
> b8c25112d407        nat                 nat                 local
> 9331cc488dfb        none                null                local

```
ipconfig
```

> ...

SOOOOOOOOOOOOOOO: looks like the docker network create command gets me halfway
there, but unfortunately in the script the New-HNSNetwork is followed by a
New-HnsEndpoint and an Attach-HnsHostEndpoint and a netsh command. Booo.

This is what I have after running start-kubelet.ps1 on a third worker node (1709):

```
Get-Content C:\k\cni\config\l2bridge.conf
```

> {
>     "cniVersion":  "0.2.0",
>     "name":  "l2bridge",
>     "type":  "wincni.exe",
>     "master":  "Ethernet",
>     "capabilities":  {
>                          "portMappings":  true
>                      },
>     "ipam":  {
>                  "environment":  "azure",
>                  "subnet":  "10.200.3.0/24",
>                  "routes":  [
>                                 {
>                                     "GW":  "10.200.3.2"
>                                 }
>                             ]
>              },
>     "dns":  {
>                 "Nameservers":  [
>                                     "10.32.0.10"
>                                 ],
>                 "Search":  [
>                                "cluster.local"
>                            ]
>             },
>     "AdditionalArgs":  [
>                            {
>                                "Name":  "EndpointPolicy",
>                                "Value":  {
>                                              "Type":  "OutBoundNAT",
>                                              "ExceptionList":  [
>                                                                    "10.200.0.0/16",
>                                                                    "10.32.0.0/24",
>                                                                    "10.240.0.0/24"
>                                                                ]
>                                          }
>                            },
>                            {
>                                "Name":  "EndpointPolicy",
>                                "Value":  {
>                                              "Type":  "ROUTE",
>                                              "DestinationPrefix":  "10.32.0.0/24",
>                                              "NeedEncap":  true
>                                          }
>                            },
>                            {
>                                "Name":  "EndpointPolicy",
>                                "Value":  {
>                                              "Type":  "ROUTE",
>                                              "DestinationPrefix":  "10.240.0.23/32",
>                                              "NeedEncap":  true
>                                          }
>                            }
>                        ]
> }

```
ipconfig
```

> Ethernet adapter vEthernet (Ethernet) 2:
>
>    Connection-specific DNS Suffix  . : c.peterhornyack-prod-no-enforcer.internal
>    Link-local IPv6 Address . . . . . : fe80::d0b7:5c4d:5a57:a2c2%8
>    IPv4 Address. . . . . . . . . . . : 10.240.0.23
>    Subnet Mask . . . . . . . . . . . : 255.255.255.0
>    Default Gateway . . . . . . . . . : 10.240.0.1
>
> Ethernet adapter vEthernet (cbr0):
>
>    Connection-specific DNS Suffix  . :
>    Link-local IPv6 Address . . . . . : fe80::31c0:c008:1bba:866e%23
>    IPv4 Address. . . . . . . . . . . : 10.200.3.2
>    Subnet Mask . . . . . . . . . . . : 255.255.255.0
>    Default Gateway . . . . . . . . . :
>
> Ethernet adapter vEthernet (nat):
>
>    Connection-specific DNS Suffix  . :
>    Link-local IPv6 Address . . . . . : fe80::501d:5034:e2f6:bf9a%3
>    IPv4 Address. . . . . . . . . . . : 172.17.144.1
>    Subnet Mask . . . . . . . . . . . : 255.255.240.0
>    Default Gateway . . . . . . . . . :

Then on worker-3-1709 I tried running start-kubelet.ps1 followed by start-kubeproxy.ps1 as described in Microsoft steps. Doesn't seem to connect to cluster though - here's the output from start-kubelet that repeats over and over:

> I0629 23:44:52.443372    3672 config.go:99] Looking for [api], have seen map[]
> I0629 23:44:52.489250    3672 generic.go:183] GenericPLEG: Relisting
> I0629 23:44:52.500934    3672 reflector.go:240] Listing and watching *v1.Pod from k8s.io/kubernetes/pkg/kubelet/config/apiserver.go:47
> I0629 23:44:52.502901    3672 reflector.go:240] Listing and watching *v1.Node from k8s.io/kubernetes/pkg/kubelet/kubelet.go:461
> I0629 23:44:52.502901    3672 round_trippers.go:405] GET https://35.230.52.90:6443/api/v1/pods?fieldSelector=spec.nodeName%3Dworker-3-1709&limit=500&resourceVersion=0 403 Forbidden in 1 milliseconds
> E0629 23:44:52.503873    3672 reflector.go:205] k8s.io/kubernetes/pkg/kubelet/config/apiserver.go:47: Failed to list *v1.Pod: pods is forbidden: User "system:kube-proxy" cannot list pods at the cluster scope
> I0629 23:44:52.503873    3672 round_trippers.go:405] GET https://35.230.52.90:6443/api/v1/nodes?fieldSelector=metadata.name%3Dworker-3-1709&limit=500&resourceVersion=0 403 Forbidden in 0 milliseconds
> E0629 23:44:52.503873    3672 reflector.go:205] k8s.io/kubernetes/pkg/kubelet/kubelet.go:461: Failed to list *v1.Node: nodes is forbidden: User "system:kube-proxy" cannot list nodes at the cluster scope
> I0629 23:44:52.541962    3672 config.go:99] Looking for [api], have seen map[]

Wtf? Did I not set up the auth correctly?

This looks concerning from start-kubeproxy:

> W0629 23:42:35.803553    3388 server.go:601] Failed to retrieve node info: nodes "worker-3-1709" not found
> W0629 23:42:35.804527    3388 proxier.go:457] invalid nodeIP, initializing kube-proxy with 127.0.0.1 as nodeIP

Also (not sure if this matters, feel like I saw it in the past too):

> W0629 23:42:35.807480    3388 proxier.go:462] clusterCIDR not specified, unable to distinguish between internal and external traffic

Hmmm, the "cannot list pods at the cluster scope" thing might be related to "roles" and RBAC: https://github.com/kubernetes/kubernetes/issues/45359 for example. https://github.com/kubernetes/kubernetes/issues/58882#issuecomment-361161573 . 

Went back to lab 08 and did RBAC steps on controller-0: didn't help, start-kubelet and start-kubeproxy still emit same logs.

Went back to lab 08 and changed apiserver --authorization-mode from Node,RBAC to AlwaysAllow (https://kubernetes.io/docs/reference/access-authn-authz/rbac/), then:
  sudo systemctl daemon-reload
  sudo systemctl stop kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
SUCCESS!
  $ kubectl get nodes
  NAME            STATUS     ROLES     AGE       VERSION
  worker-0        NotReady   <none>    5d        v1.10.5
  worker-3-1709   Ready      <none>    21s       v1.10.3

ORIGINAL l2bridge.conf:
```
{
    "cniVersion":  "0.2.0",
    "name":  "l2bridge",
    "type":  "wincni.exe",
    "master":  "Ethernet",
    "capabilities":  {
        "portMappings":  true
    },
    "ipam":  {
        "environment":  "azure",
        "subnet":  "192.168.1.0/24",
        "routes":  [
            {
                "GW":  "192.168.1.2"
            }
        ]
    },
    "dns":  {
        "Nameservers":  [
            "11.0.0.10"
        ],
        "Search": [
            "svc.cluster.local"
        ]
    },
    "AdditionalArgs":  [
        {
            "Name":  "EndpointPolicy",
            "Value":  {
                "Type":  "OutBoundNAT",
                "ExceptionList":  [
                    "192.168.0.0/16",
                    "11.0.0.0/8",
                    "10.124.24.0/23"
                ]
            }
        },
        {
            "Name":  "EndpointPolicy",
            "Value":  {
                "Type":  "ROUTE",
                "DestinationPrefix":  "11.0.0.0/8",
                "NeedEncap":  true
            }
        },
        {
            "Name":  "EndpointPolicy",
            "Value":  {
                "Type":  "ROUTE",
                "DestinationPrefix":  "10.124.24.196/32",
                "NeedEncap":  true
            }
        }
    ]
}
```


