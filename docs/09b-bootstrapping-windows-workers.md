# Bootstrapping the Windows Worker Nodes

In this lab you will bootstrap two Windows Kubernetes worker nodes.  The
following components will be installed on each node: [container networking
plugins](https://github.com/containernetworking/cni),
[kubectl](https://kubernetes.io/docs/reference/kubectl/overview/), and
[kubelet](https://kubernetes.io/docs/admin/kubelet), and
[kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

Docker should be preinstalled if you're using the Windows-for-containers GCE
image. If you're using a different Windows image, follow [these
steps](https://cloud.google.com/compute/docs/containers/#install_docker) to
install Docker and configure some GCE-specific workarounds.

## Prerequisites

The commands in this lab must be run on each worker instance: `worker-1` and
`worker-2`. RDP to each instance using your RDP client as described previously,
then run the commands below in a PowerShell session.

TODO: figure out how to invoke these commands remotely.

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. See the [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) section in the Prerequisites lab.

## Provisioning a Kubernetes Worker Node

## Install additional software

```
Install-Package -Force 7Zip4Powershell
```

Disable the Windows firewall:

```
Set-NetFirewallProfile -Profile Domain, Public, Private -Enabled False
```

Install VS Code for editing text files (instead of notepad):
```
Invoke-WebRequest https://raw.githubusercontent.com/PowerShell/vscode-powershell/master/scripts/Install-VSCode.ps1 -OutFile C:\Install-VSCode.ps1
.\Install-VSCode.ps1
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\Program Files\Microsoft VS Code", [EnvironmentVariableTarget]::Machine)
# Need to restart to take effect though :(
& 'C:\Program Files\Microsoft VS Code\Code.exe' C:\file\to\edit.txt
```

### Create the "pause image"

[These
steps](https://github.com/MicrosoftDocs/Virtualization-Documentation/blob/live/virtualization/windowscontainers/kubernetes/getting-started-kubernetes-windows.md#creating-the-pause-image)
are copied from Microsoft's Virtualization-Documentation, with some
modifications.

```
mkdir C:\k\pauseimage
cd C:\k\pauseimage
New-Item -ItemType file Dockerfile
notepad.exe Dockerfile
```

Paste these contents into the Dockerfile, then save and close.

```
FROM microsoft/nanoserver:1803

CMD cmd /c ping -t localhost
```

TODO: make the 1709, 1803 version tag a constant / metadata tag.

TODO: figure out how to write this content to the Dockerfile from the command line.

```
docker build -t kubeletwin/pause .
```

After the container builds, run the following command to confirm that it works:

```
docker run kubeletwin/pause
```

After a short delay you should see continuous pings to and replies from ::1.
Press `Ctrl-C` to break out of the container.

### Download and Install Worker Binaries

[These
steps](https://github.com/MicrosoftDocs/Virtualization-Documentation/blob/live/virtualization/windowscontainers/kubernetes/getting-started-kubernetes-windows.md#downloading-binaries)
are borrowed from Microsoft's Virtualization-Documentation, with some
modifications.

Run the following commands to download the Kubernetes node binaries for the
release of your choice. These commands use
[1.10.5](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.10.md#downloads-for-v1105).
You can use the [Google Cloud Storage
browser](https://console.cloud.google.com/storage/browser/kubernetes-release/release/v1.10.5/bin/windows/amd64/)
to find the appropriate path for other releases.

TODO: is kubeadm required?

```
mkdir C:\k\node
cd C:\k\node

# Disable progress bar to dramatically increase download speed.
$ProgressPreference = 'SilentlyContinue'
Invoke-WebRequest https://storage.googleapis.com/kubernetes-release/release/v1.10.5/bin/windows/amd64/kubeadm.exe -OutFile kubeadm.exe
Invoke-WebRequest https://storage.googleapis.com/kubernetes-release/release/v1.10.5/bin/windows/amd64/kubectl.exe -OutFile kubectl.exe
Invoke-WebRequest https://storage.googleapis.com/kubernetes-release/release/v1.10.5/bin/windows/amd64/kubelet.exe -OutFile kubelet.exe
Invoke-WebRequest https://storage.googleapis.com/kubernetes-release/release/v1.10.5/bin/windows/amd64/kube-proxy.exe -OutFile kube-proxy.exe
```

TODO: try containerd and see if it works?

Add the node binary directory to your path. Note that the updated path will not
take effect in new terminals until after a reboot.

```
$env:Path += ";C:\k\node"
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\k\node", [EnvironmentVariableTarget]::Machine)
```

### Create the Kubernetes config

Update the KUBECONFIG environment variable to point to the kubeconfig that we created previously.

```
$env:KUBECONFIG="C:\k\kube-proxy.kubeconfig"
[Environment]::SetEnvironmentVariable("KUBECONFIG", "C:\k\kube-proxy.kubeconfig", [EnvironmentVariableTarget]::User)
```

(This config can also be specified using the `--kubeconfig` flag for `kubectl`).

### Verification

Run these commands to verify that kubectl is properly using the kubeconfig:

```
kubectl config view
```

> apiVersion: v1
> clusters:
> - cluster:
>     certificate-authority-data: REDACTED
>     server: https://35.230.52.90:6443
>   name: kubernetes-the-hard-way
> contexts:
> - context:
>     cluster: kubernetes-the-hard-way
>     user: system:kube-proxy
>   name: default
> current-context: default
> kind: Config
> preferences: {}
> users:
> - name: system:kube-proxy
>   user:
>     client-certificate-data: REDACTED
>     client-key-data: REDACTED

```
kubectl version
```

> Client Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.5", GitCommit:"32ac1c9073b132b8ba18aa830f46b77dcceb0723", GitTreeState:"clean", BuildDate:"2018-06-21T11:46:00Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"windows/amd64"}
> Server Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.5", GitCommit:"32ac1c9073b132b8ba18aa830f46b77dcceb0723", GitTreeState:"clean", BuildDate:"2018-06-21T11:34:22Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/amd64"}

If you see `Unable to connect to the server: dial tcp [::1]:8080: connectex: No
connection could be made because the target machine actively refused it.` then
the kubeconfig is not being read properly or was not set up properly in the
previous labs.

### Configure CNI Networking

TODO: at this point the Microsoft guide has these
[contents](https://github.com/Microsoft/SDN/tree/master/Kubernetes/windows)
under C:\k:

 * A bunch of .ps1 scripts that I probably don't need.
 * cni/wincni.exe
 * cni/config/l2bridge.conf
 * debug/ - has scripts for capturing debug traces.

How is wincni.exe used? What does l2bridge.conf do?

First attempt: put cni/wincni.exe into place, then manually paste in
l2bridge.conf, using values that start-kubelet.ps1 would put there! This is
pretty similar to how the Linux worker node is set up.

The wincni repository is [here](https://github.com/Microsoft/SDN/tree/master/Kubernetes/wincni), but it's unclear if/how that code can be built (just a `go build` command?). For now, we'll use the prebuilt wincni.exe binary:

```
mkdir C:\k\cni

# Need to specify TLS version 1.2 since GitHub API requires it
[Net.ServicePointManager]::SecurityProtocol = [Net.ServicePointManager]::SecurityProtocol -bor [Net.SecurityProtocolType]::Tls12
Invoke-WebRequest https://github.com/Microsoft/SDN/raw/master/Kubernetes/windows/cni/wincni.exe -OutFile C:\k\cni\wincni.exe
```

TODO: what does -bor do? Seems to work at least, resolves error
`Invoke-WebRequest : The request was aborted: Could not create SSL/TLS secure
channel.`.

```
mkdir C:\k\cni\config
New-Item -ItemType file C:\k\cni\config\l2bridge.conf
notepad.exe C:\k\cni\config\l2bridge.conf
```

TODO: figure out how to insert contents directly from powershell. Look at
previous tests.

TODO: see 09a-sidebar.md for debugging I did at this point.

TODO: figure out how to get $podCidr from metadata server and then use /
transform it for the steps below. e.g.:
```
POD_CIDR=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/pod-cidr)
```

Paste the following contents into the file (for worker-1!):

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
        "subnet":  "10.200.1.0/24",
        "routes":  [
            {
                "GW":  "10.200.1.2"
            }
        ]
    },
    "dns":  {
        "Nameservers":  [
            "10.32.0.10"
        ],
        "Search": [
            "cluster.local"
        ]
    },
    "AdditionalArgs":  [
        {
            "Name":  "EndpointPolicy",
            "Value":  {
                "Type":  "OutBoundNAT",
                "ExceptionList":  [
                    "10.200.0.0/16",
                    "10.32.0.0/24",
                    "10.240.0.0/24"
                ]
            }
        },
        {
            "Name":  "EndpointPolicy",
            "Value":  {
                "Type":  "ROUTE",
                "DestinationPrefix":  "10.32.0.0/24",
                "NeedEncap":  true
            }
        },
        {
            "Name":  "EndpointPolicy",
            "Value":  {
                "Type":  "ROUTE",
                "DestinationPrefix":  "10.240.0.23/32",
                "NeedEncap":  true
            }
        }
    ]
}
```

Explanation of the fields in the CNI config:

* name: "l2bridge" is the type of container networking that will be used. Unsure
  if it's meaningful to wincni.exe. Possible / likely values are: [ICS,
  Internal, Transparent, NAT, Overlay, L2Bridge, L2Tunnel, Layered, Private].
  (see hns.psm1: New-HnsNetwork).
* master: the name of the primary network interface, presumably. In my GCE
  instance it is also Ethernet.
* capabilities.portMappings: unsure.
* ipam.environment: obviously we're not using azure, not sure it matters.
* ipam.subnet: the $podCIDR - 192.168.1.0/24 in Microsoft steps, 10.200.N.0/24
  for worker-N in these steps.
    - Get using curl from metadata server!
* ipam.routes.GW: the .2 address in the $podCIDR (Windows nodes use .2 for the
  gateway due to a platform limitation). So, for worker-N in these steps, should
  be 10.200.N.2.
    - ipam probably happens on a per-pod basis, and containers in the pod
      communicate with the .2 address for the gateway (whereas the host side
      of the gateway is the .1), I think.
* dns.Nameservers: contains $KubeDnsServiceIp. For Microsoft steps this is currently
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
appropriately. See article that Alain sent.

Then, we need to invoke some commands to set up the Windows node networking.
First, get the Microsoft helper scripts:

```
cd C:\k
Invoke-WebRequest https://github.com/Microsoft/SDN/archive/master.zip -OutFile C:\k\Microsoft-SDN-master.zip
Expand-Archive C:\k\Microsoft-SDN-master.zip -DestinationPath C:\k
rm C:\k\Microsoft-SDN-master.zip
mv C:\k\SDN-master\Kubernetes\windows\ C:\k\Microsoft-SDN-scripts
rm -Recurse -Force C:\k\SDN-master\
```

TODO: instead of using these unsupported scripts that Microsoft ad-hoc released,
write your own scripts / executables that build on hcsshim instead. Just check
these scripts into this repository.

Then run the following commands to configure the Host Networking Service (HNS).
Note that your RDP session may be interrupted when you invoke these commands.

```
$podCidr = "10.200.1.0/24"
$podGateway = "10.200.1.1"
$podEndpointGateway = "10.200.1.2"
$hnsNetworkName = "l2bridge"
$endpointName = "cbr0"
$vnicName = "vEthernet ($endpointName)"

Import-Module C:\k\Microsoft-SDN-scripts\hns.psm1

New-HNSNetwork -Type "L2Bridge" -AddressPrefix $podCidr -Gateway $podGateway -Name $hnsNetworkName -Verbose
$hnsNetwork = Get-HnsNetwork | ? Type -EQ "L2Bridge"
$hnsEndpoint = New-HnsEndpoint -NetworkId $hnsNetwork.Id -Name $endpointName -IPAddress $podEndpointGateway -Gateway "0.0.0.0" -Verbose
Attach-HnsHostEndpoint -EndpointID $hnsEndpoint.Id -CompartmentID 1 -Verbose
netsh interface ipv4 set interface "$vnicName" forwarding=enabled
Get-HNSPolicyList | Remove-HnsPolicyList
```

TODO: for Linux at this point we configure containerd and gVisor. Do I need to
do anything special for configuring Docker for Windows nodes?
TODO: also not configuring here: loopback network?

### Configure the Kubelet

TODO: skipped these steps on Windows node for now since I enabled anonymous /
AlwaysAllow authentication. Where should these files be placed on Windows?

```
{
  sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
  sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
  sudo mv ca.pem /var/lib/kubernetes/
}
```

Create the `kubelet-config.yaml` configuration file:

TODO: do other Windows demos even have a "kubelet-config"??

TODO: for authentication set anonymous to true instead of false since I skipped
RBAC setup (webhook). For authorization set mode to AlwaysAllow instead of
Webhook
(https://github.com/kubernetes/kubernetes/blob/cd78e999f9aade259c177c1698129311c83aa7d3/pkg/kubeapiserver/authorizer/modes/modes.go#L30).

```
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: true
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: AlwaysAllow
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF
```

### Configure the Kubernetes Proxy

TODO: skipped these steps for now. Where do these files go on a Windows node?

```
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

Create the `kube-proxy-config.yaml` configuration file:

```
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

### Start the Worker Services


Finally, open a separate powershell window and start the kubelet:

```
$KubeDnsServiceIp = "10.32.0.10"
C:\k\node\kubelet.exe --hostname-override=$(hostname) --v=6 `
  --pod-infra-container-image=kubeletwin/pause --resolv-conf="" `
  --allow-privileged=true --enable-debugging-handlers `
  --cluster-dns=$KubeDnsServiceIp --cluster-domain=cluster.local `
  --kubeconfig=C:\k\kube-proxy.kubeconfig --hairpin-mode=promiscuous-bridge `
  --image-pull-progress-deadline=20m --cgroups-per-qos=false `
  --enforce-node-allocatable="" --network-plugin=cni --cni-bin-dir="C:\k\cni" `
  --cni-conf-dir "C:\k\cni\config"
```

Note: to use Hyper-V isolation for the pods add
`--feature-gates=HyperVContainer=true`.

Wait a few seconds, then open a new powershell window and then start kube-proxy:

```
$hnsNetworkName = "l2bridge"
$env:KUBE_NETWORK = $hnsNetworkName
C:\k\node\kube-proxy.exe --v=4 --proxy-mode=kernelspace --hostname-override=$(hostname) --kubeconfig=C:\k\kube-proxy.kubeconfig
```

After a short delay the worker nodes should successfully join the cluster:

> PS C:\k> kubectl get nodes
> NAME            STATUS    ROLES     AGE       VERSION
> worker-0        Ready     <none>    5d        v1.10.5
> worker-1        Ready     <none>    1m        v1.10.5

## Verification

> The compute instances created in this tutorial will not have permission to complete this section. Run the following commands from the same machine used to create the compute instances.

List the registered Kubernetes nodes:

```
gcloud compute ssh controller-0 \
  --command "kubectl get nodes --kubeconfig admin.kubeconfig"
```

> output

```
NAME            STATUS    ROLES     AGE       VERSION
worker-0        Ready     <none>    20m       v1.10.5
worker-1        Ready     <none>    2m        v1.10.5
worker-2        Ready     <none>    2m        v1.10.5
```

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
