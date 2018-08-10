# Bootstrapping the Windows Worker Nodes

In this lab you will bootstrap two Windows Kubernetes worker nodes.  The
following components will be installed on each node: [container networking
plugins](https://github.com/containernetworking/cni),
[kubectl](https://kubernetes.io/docs/reference/kubectl/overview/), and
[kubelet](https://kubernetes.io/docs/admin/kubelet), and
[kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

Docker should be preinstalled on your Windows instances if you created them
using the `windows-1803-core-for-containers` image as instructed. If you're
using a different Windows image, follow [these
steps](https://cloud.google.com/compute/docs/containers/#install_docker) to
install Docker and configure some GCE-specific workarounds.

## Prerequisites

The commands in this lab must be run on each Windows worker instance: `worker-1`
and `worker-2`. RDP to each instance using your RDP client as described
previously, then run the commands below in a PowerShell session.

Disable the Windows firewall (NOTE: this is currently recommended while
Kubernetes Windows support is beta, but should not be done for production
clusters):

```
Set-NetFirewallProfile -Profile Domain, Public, Private -Enabled False
```

### Install additional software

[Chocolatey](https://chocolatey.org/about) is a package manager for Windows.
Follow the [Installing
Chocolatey](https://chocolatey.org/install#installing-chocolatey) steps to
install it using PowerShell.

Once Chocolatey is installed, use it to install Git, 7Zip and Notepad++:

```
choco install -y git 7zip notepadplusplus
```

After installing this software, a reboot is recommended to ensure that the
executables will be on your PATH for all your PowerShell instances:

TODO: set up permanent environment variables at this step, so they'll be
available to all subsequent PowerShell instances?

```
Restart-Computer -Force
```

### Create the "pause image"

[These
steps](https://github.com/MicrosoftDocs/Virtualization-Documentation/blob/live/virtualization/windowscontainers/kubernetes/getting-started-kubernetes-windows.md#creating-the-pause-image)
are copied from Microsoft's Virtualization-Documentation, with some
modifications.

The "pause image" simply runs the ping command indefinitely. Create a Dockerfile
for it:

```
$k8sDir = "C:\k8s_hardway"
$pauseImage = "${k8sDir}\pauseimage"
mkdir ${PAUSE_IMAGE}
New-Item -ItemType file ${pauseImage}\Dockerfile
Set-Content ${pauseImage}\Dockerfile "FROM microsoft/nanoserver:1803`n`nCMD cmd /c ping -t localhost"
```

TODO: make the 1709, 1803 version tag a constant / metadata tag.

Then build the container image:

```
docker build -t kubeletwin/pause ${pauseImage}
```

After the container builds, run the following command to confirm that it works:

```
docker run kubeletwin/pause
```

After a short delay you should see continuous pings to and replies from ::1.
Press `Ctrl-C` to break out of the container.

### Download and Install Kubernetes Binaries

[These
steps](https://github.com/MicrosoftDocs/Virtualization-Documentation/blob/live/virtualization/windowscontainers/kubernetes/getting-started-kubernetes-windows.md#downloading-binaries)
are borrowed from Microsoft's Virtualization-Documentation, with some
modifications.

Run the following commands to download the Kubernetes node binaries for the
version we specified earlier as a project-level metadata value.

```
$k8sVersion = "$(gcloud compute project-info describe `
  --format='value(commonInstanceMetadata.items.k8s-version)')"
$nodeDir = "${k8sDir}\node"

mkdir ${nodeDir}

# Disable progress bar to dramatically increase download speed.
$ProgressPreference = 'SilentlyContinue'
Invoke-WebRequest `
  https://storage.googleapis.com/kubernetes-release/release/${k8sVersion}/bin/windows/amd64/kubectl.exe `
  -OutFile ${nodeDir}\kubectl.exe
Invoke-WebRequest `
  https://storage.googleapis.com/kubernetes-release/release/${k8sVersion}/bin/windows/amd64/kubelet.exe `
  -OutFile ${nodeDir}\kubelet.exe
Invoke-WebRequest `
  https://storage.googleapis.com/kubernetes-release/release/${k8sVersion}/bin/windows/amd64/kube-proxy.exe `
  -OutFile ${nodeDir}\kube-proxy.exe
```

TODO: try containerd and see if it works.

Add the node binary directory to your path. Note that the updated path will not
take effect in new terminals until after a reboot.

```
$env:Path += ";${nodeDir}"
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";${nodeDir}", `
  [EnvironmentVariableTarget]::Machine)
```

### Create the Kubernetes config

Update the KUBECONFIG environment variable to point to the kubeconfig that we
created previously.

TODO: who/what uses this? Can I skip it? There are different kubeconfigs for
kubelet and kube-proxy, if I need to use one here which one should it be?

```
$env:KUBECONFIG="${k8sDir}\$(hostname).kubeconfig"
[Environment]::SetEnvironmentVariable("KUBECONFIG", `
  "${k8sDir}\$(hostname).kubeconfig", [EnvironmentVariableTarget]::User)
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
>     user: system:node:wworker-2
>   name: default
> current-context: default
> kind: Config
> preferences: {}
> users:
> - name: system:node:wworker-2
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
under ${k8sDir}:

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
$cniDir = "${k8sDir}\cni"
mkdir ${cniDir}

# Need to specify TLS version 1.2 since GitHub API requires it; without changing
# the SecurityProtocol, Invoke-WebRequest will fail with "The request was
# aborted: Could not create SSL/TLS secure channel."
#
# "-bor" in this command is for bitwise-OR:
[Net.ServicePointManager]::SecurityProtocol = `
  [Net.ServicePointManager]::SecurityProtocol `
  -bor [Net.SecurityProtocolType]::Tls12

Invoke-WebRequest `
  https://github.com/Microsoft/SDN/raw/master/Kubernetes/windows/cni/wincni.exe `
  -OutFile ${cniDir}\wincni.exe
```

Now create the configuration file for "l2bridge" mode for the CNI plugin. (NOTE:
if you need to edit the configuration or correct a mistake you can open the file
using `notepad++.exe`.)

```
$podCidr = "$([System.Text.Encoding]::ASCII.GetString((`
  Invoke-WebRequest -UseBasicParsing -H @{'Metadata-Flavor' = 'Google'} `
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/pod-cidr).Content))"

# For Windows nodes the pod gateway IP address is the .1 address in the pod
# CIDR for the host, but from inside containers it's the .2 address.
$podGateway = ${podCidr}.substring(0, ${podCidr}.lastIndexOf('.')) + '.1'
$podEndpointGateway = ${podCidr}.substring(0, ${podCidr}.lastIndexOf('.')) + '.2'

mkdir ${cniDir}\config
$l2bridgeConf = "${cniDir}\config\l2bridge.conf"
New-Item -ItemType file ${l2bridgeConf}

Set-Content ${l2bridgeConf} `
  '{
    "cniVersion":  "0.2.0",
    "name":  "l2bridge",
    "type":  "wincni.exe",
    "master":  "Ethernet",
    "capabilities":  {
        "portMappings":  true
    },
    "ipam":  {
        "environment":  "azure",
        "subnet":  "POD_CIDR",
        "routes":  [
            {
                "GW":  "POD_ENDPOINT_GW"
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
}'.replace('POD_CIDR', ${podCidr}).replace('POD_ENDPOINT_GW', ${podEndpointGateway})
```

TODO: the 10.240.0.23 address needs adjusting, I think: should be e.g.
10.240.0.21, 10.240.0.22 for worker-1, worker-2. Right?

NOTE: see troubleshooting/09a-sidebar.md for debugging I did at this point.

Explanation of the fields in the CNI config:

* name: "l2bridge" is the type of container networking that will be used. Unsure
  if it's meaningful to wincni.exe. Possible / likely values are: [ICS,
  Internal, Transparent, NAT, Overlay, L2Bridge, L2Tunnel, Layered, Private].
  (see hns.psm1: New-HnsNetwork).
* master: the name of the primary network interface, presumably. In my GCE
  instance it is also Ethernet.
* capabilities.portMappings: unsure.
* ipam.environment: obviously we're not using azure, not sure it matters.
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
appropriately. See article.

Then, we need to invoke some commands to set up the Windows node networking.
First, get the Microsoft helper scripts:

```
git clone https://github.com/Microsoft/SDN.git ${k8sDir}\SDN
$sdnScripts = "${k8sDir}\SDN\Kubernetes\windows"
```

TODO: instead of using these unsupported scripts that Microsoft ad-hoc released,
write your own scripts / executables that build on hcsshim instead. Just check
these scripts into this repository.

Then run the following commands to configure the Host Networking Service (HNS).
Note that your RDP session may be interrupted when you invoke these commands.

```
$hnsNetworkName = "l2bridge"
$endpointName = "cbr0"
$vnicName = "vEthernet ($endpointName)"

Import-Module ${sdnScripts}\hns.psm1

New-HNSNetwork -Type "L2Bridge" -AddressPrefix $podCidr -Gateway $podGateway `
  -Name $hnsNetworkName -Verbose
$hnsNetwork = Get-HnsNetwork | ? Type -EQ "L2Bridge"
$hnsEndpoint = New-HnsEndpoint -NetworkId $hnsNetwork.Id -Name $endpointName `
  -IPAddress $podEndpointGateway -Gateway "0.0.0.0" -Verbose
Attach-HnsHostEndpoint -EndpointID $hnsEndpoint.Id -CompartmentID 1 -Verbose
netsh interface ipv4 set interface "$vnicName" forwarding=enabled
Get-HNSPolicyList | Remove-HnsPolicyList
```

TODO: for Linux at this point we configure containerd and gVisor. Do I need to
do anything special for configuring Docker for Windows nodes?
TODO: also not configuring here: loopback network?

### Configure the Kubelet

TODO: C:\k8s_hardway still has these files:
  ca.pem
  kube-proxy.kubeconfig
  worker-2-key.pem
  worker-2.kubeconfig
  worker-2.pem
Just leave them where they are for now, but consider moving to a better place?

Create the `kubelet-config.yaml` configuration file (Note: for authentication
set anonymous to true instead of false since I skipped RBAC setup (webhook). For
authorization set mode to AlwaysAllow instead of Webhook
(https://github.com/kubernetes/kubernetes/blob/cd78e999f9aade259c177c1698129311c83aa7d3/pkg/kubeapiserver/authorizer/modes/modes.go#L30)).

```
$kubeletConfig = "${k8sDir}\kubelet-config.yaml"

Set-Content ${kubeletConfig} `
'kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: true
  webhook:
    enabled: true
  x509:
    clientCAFile: "K8S_DIR\ca.pem"
authorization:
  mode: AlwaysAllow
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "POD_CIDR"
runtimeRequestTimeout: "15m"
tlsCertFile: "K8S_DIR\HOSTNAME.pem"
tlsPrivateKeyFile: "K8S_DIR\HOSTNAME-key.pem"'.replace('K8S_DIR', ${k8sDir}).replace('POD_CIDR', ${podCidr}).replace('HOSTNAME', $(hostname))
```

### Configure the Kubernetes Proxy

TODO: skipped these steps for now. Create a kube-proxy-config on the Windows
node and move the kubeproxy command-line flags from below to here.

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

TODO: make $k8sDir / $nodeDir part of the environment that sticks across
powershells!
  Also: $env:KUBECONFIG ?

```
$k8sDir = "C:\k8s_hardway"
$nodeDir = "${k8sDir}\node"
$cniDir = "${k8sDir}\cni"
$kubeletConfig = "${k8sDir}\kubelet-config.yaml"
$kubeDnsServiceIp = "10.32.0.10"

& ${nodeDir}\kubelet.exe --hostname-override=$(hostname) --v=6 `
  --pod-infra-container-image=kubeletwin/pause --resolv-conf="" `
  --allow-privileged=true --config=${kubeletConfig} `
  --enable-debugging-handlers `
  --kubeconfig=${k8sDir}\$(hostname).kubeconfig --hairpin-mode=promiscuous-bridge `
  --image-pull-progress-deadline=20m --cgroups-per-qos=false `
  --enforce-node-allocatable="" --network-plugin=cni --cni-bin-dir="${cniDir}" `
  --cni-conf-dir "${cniDir}\config" --register-node=true
```

Note: removed these command-line flags since they are now part of
kubelet-config.yaml:
*   --cluster-dns=${kubeDnsServiceIp}
*   --cluster-domain=cluster.local

Note: the Linux worker's kubelet is also invoked with these flags, all of which
are either already present above or which should be omitted for the Windows
kubelet:
*   --container-runtime=remote \\
*   --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
*   --image-pull-progress-deadline=2m \\
*   --kubeconfig=/var/lib/kubelet/kubeconfig \\
*   --network-plugin=cni \\
*   --register-node=true \\
*   --v=2

TODO: move the rest of the kubelet command-line flags into kubelet-config.yaml.
See https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/ for
the flags that are supported.

Note: to use Hyper-V isolation for the pods add
`--feature-gates=HyperVContainer=true`.

Wait a few seconds, then open a new powershell window and then start kube-proxy:

```
$k8sDir = "C:\k8s_hardway"
$nodeDir = "${k8sDir}\node"
$hnsNetworkName = "l2bridge"
$env:KUBE_NETWORK = $hnsNetworkName

& ${nodeDir}\kube-proxy.exe --v=4 --proxy-mode=kernelspace `
  --hostname-override=$(hostname) --kubeconfig=${k8sDir}\kube-proxy.kubeconfig `
  --cluster-cidr="10.200.0.0/16"
```

TODO: added --cluster-cidr flag to match Linux kube-proxy-config.yaml. Does the
Windows kube-proxy still work? Seems to.

After a short delay the worker nodes should successfully join the cluster:

```
$k8sDir = "C:\k8s_hardway"
$nodeDir = "${k8sDir}\node"
& $k8sDir\node\kubectl get nodes --kubeconfig "${k8sDir}\kube-proxy.kubeconfig"
```

> Output:

```
NAME            STATUS    ROLES     AGE       VERSION
worker-0        Ready     <none>    1h        v1.10.5
worker-1        Ready     <none>    1m        v1.10.5
worker-2        Ready     <none>    1m        v1.10.5
```

NOTE: see troubleshooting/windows-worker-cluster-join-RBAC.md for notes about
issues I was having with getting Windows nodes to join when the apiserver was
started with Node,RBAC authorization.

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
worker-0        Ready     <none>    1h        v1.10.5
worker-1        Ready     <none>    1m        v1.10.5
worker-2        Ready     <none>    1m        v1.10.5
```

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
