# Explanation of the Windows CNI configuration fields

Explanation of the fields in the CNI config (`l2bridge.conf`):

* name: "l2bridge" is the type of container networking that will be used. Unsure
  if it's meaningful to wincni.exe. Possible / likely values are: [ICS,
  Internal, Transparent, NAT, Overlay, L2Bridge, L2Tunnel, Layered, Private].
  (see hns.psm1: New-HnsNetwork).
* master: the name of the primary network interface, presumably. In my GCE
  instance it is also Ethernet.
* capabilities.portMappings: unsure.
* ipam.environment: obviously we're not using azure, not sure it matters.
  Discussion on Kubernetes #sig-windows Slack channel suggests that the ipam
  block can be omitted entirely.
* ipam.subnet: the $podCidr - 192.168.1.0/24 in Microsoft steps, 10.200.N.0/24
  for worker-N in these steps.
    - Get using curl from metadata server!
* ipam.routes.GW: the .2 address in the $podCidr (Windows nodes use .2 for the
  gateway due to a platform limitation). So, for worker-N in these steps, should
  be 10.200.N.2.
    - ipam probably happens on a per-pod basis, and containers in the pod
      communicate with the .2 address for the gateway (whereas the host side
      of the gateway is the .1), I think.
* dns.Nameservers: contains $kubeDnsServiceIp. For Microsoft steps this is currently
  hardcoded to 11.0.0.10.
    - In these steps it seems to be "10.32.0.10" (see kubelet-config.yaml for Linux
      node). 10.32.0.0/24 is the service-cluster-ip-range (see lab 8), .10 seems
      to be the canonical DNS IP within that subnet.
* dns.Search: contains $KubeDnsSuffix; "svc.cluster.local" in Microsoft steps.
    - For GCE, could be just "cluster.local" (see Linux kubelet-config.yaml), or
      maybe "svc.cluster.local" is still right. Tried the former for now.
* AdditionalArgs EndpointPolicy OutBoundNAT: ExceptionList contains
  $clusterCIDR, $serviceCIDR, and Get-MgmtSubnet.
    - $clusterCIDR: "192.168.0.0/16" for Microsoft steps. "10.200.0.0/16" for
      GCE, according to Linux kube-proxy-config.yaml.
    - $serviceCIDR: "11.0.0.0/8" for Microsoft steps. "10.32.0.0/24" for GCE I
      think.
    - Get-MgmtSubnet: "10.124.24.0/23" for Microsoft steps: still uncertain
      what this is, but after running start-kubelet.ps1 on worker-3 node I
      got "10.240.0.0/24" for these GCE steps.
* AdditionalArgs EndpointPolicy ROUTEs: DestinationPrefix one contains
  $serviceCIDR, DestinationPrefix two contains Get-MgmtIpAddress/32.
    - $serviceCIDR: "11.0.0.0/8" for Microsoft steps. "10.32.0.0/24" for GCE I
      think.
    - Get-MgmtIpAddress: 10.124.24.196 for Microsoft steps. Still uncertain
      about what this actually is, but after running start-kubelet.ps1 on
      worker-3 node I got "10.240.0.23/32" for these GCE steps.

TODO: understand what the "management" network is/does and how to configure it
appropriately. See article. Is it just the GCE internal-IP network?

See [CNI config
troubleshooting](troubleshooting/win-cni-config-troubleshooting.md) for some
informal notes I took while debugging this configuration.
