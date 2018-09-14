# Bootstrapping the Windows Worker Nodes

In this lab you will bootstrap two Windows Kubernetes worker nodes.  The
following components will be installed on each node: [Windows container
networking
plugins](https://github.com/Microsoft/SDN/tree/master/Kubernetes/wincni),
[kubectl](https://kubernetes.io/docs/reference/kubectl/overview/), and
[kubelet](https://kubernetes.io/docs/admin/kubelet), and
[kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

Docker should be preinstalled on your Windows instances if you created them
using the `windows-${WIN_VERSION}-core-for-containers` image as instructed. If
you're using a different Windows image, follow [these
steps](https://cloud.google.com/compute/docs/containers/#install_docker) to
install Docker and configure some GCE-specific workarounds.

## Prerequisites

The commands in this lab must be run on each Windows worker instance: `worker-1`
and `worker-2`. RDP to each instance using your RDP client as described
previously, then run the commands below in a PowerShell session.

### Disable the Windows firewall

NOTE: this is currently recommended while Kubernetes Windows support is beta,
but should not be done for production clusters.

```
Set-NetFirewallProfile -Profile Domain, Public, Private -Enabled False
```

### Set environment variables

Setting these environment variables globally will make them available to
all the PowerShell instances in the steps below. The variables will not be
available until after the system is restarted in a subsequent step.

```
$k8sDir = "C:\k8s_hardway"
[Environment]::SetEnvironmentVariable(
    "K8S_DIR", "${k8sDir}", "Machine")
[Environment]::SetEnvironmentVariable(
    "NODE_DIR", "${k8sDir}\node", "Machine")
[Environment]::SetEnvironmentVariable(
    "Path", $env:Path + ";${k8sDir}\node", "Machine")
[Environment]::SetEnvironmentVariable(
    "CNI_DIR", "${k8sDir}\cni", "Machine")
[Environment]::SetEnvironmentVariable(
    "KUBELET_CONFIG", "${k8sDir}\kubelet-config.yaml", "Machine")
[Environment]::SetEnvironmentVariable(
    "KUBECONFIG", "${k8sDir}\$(hostname).kubeconfig", "Machine")
[Environment]::SetEnvironmentVariable(
    "KUBE_NETWORK", "l2bridge", "Machine")
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

After installing this software, reboot to ensure that the environment variables
set previously as well as the PATH for the newly installed software will be
available for all PowerShell instances:

```
Restart-Computer -Force
```

Reconnect to the instances after the reboot and continue with the following
steps.

### Create the "pause image"

[These
steps](https://github.com/MicrosoftDocs/Virtualization-Documentation/blob/live/virtualization/windowscontainers/kubernetes/getting-started-kubernetes-windows.md#creating-the-pause-image)
are copied from Microsoft's Virtualization-Documentation, with some
modifications.

The "pause image" simply runs the ping command indefinitely. Create a Dockerfile
and build the container image:

```
$winVersion = "$([System.Text.Encoding]::ASCII.GetString((`
  Invoke-WebRequest -UseBasicParsing -H @{'Metadata-Flavor' = 'Google'} `
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/win-version).Content))"

mkdir ${env:K8S_DIR}\pauseimage
New-Item -ItemType file ${env:K8S_DIR}\pauseimage\Dockerfile
Set-Content ${env:K8S_DIR}\pauseimage\Dockerfile `
  "FROM microsoft/nanoserver:${winVersion}`n`nCMD cmd /c ping -t localhost"
docker build -t kubeletwin/pause ${env:K8S_DIR}\pauseimage
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

mkdir ${env:NODE_DIR}

# Disable progress bar to dramatically increase download speed.
$ProgressPreference = 'SilentlyContinue'
Invoke-WebRequest `
  https://storage.googleapis.com/kubernetes-release/release/${k8sVersion}/bin/windows/amd64/kubectl.exe `
  -OutFile ${env:NODE_DIR}\kubectl.exe
Invoke-WebRequest `
  https://storage.googleapis.com/kubernetes-release/release/${k8sVersion}/bin/windows/amd64/kubelet.exe `
  -OutFile ${env:NODE_DIR}\kubelet.exe
Invoke-WebRequest `
  https://storage.googleapis.com/kubernetes-release/release/${k8sVersion}/bin/windows/amd64/kube-proxy.exe `
  -OutFile ${env:NODE_DIR}\kube-proxy.exe
```

### Verify the kubeconfig

The
[kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#define-clusters-users-and-contexts)
tells `kubectl` which Kubernetes cluster to connect to and how. We set the
`${env:KUBECONFIG}` variable above to the kubeconfig that we created in an
earlier lab. Run these commands to verify that `kubectl` is properly using the
kubeconfig:

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
>     user: system:node:worker-2
>   name: default
> current-context: default
> kind: Config
> preferences: {}
> users:
> - name: system:node:worker-2
>   user:
>     client-certificate-data: REDACTED
>     client-key-data: REDACTED

```
kubectl version
```

> Client Version: version.Info{Major:"1", Minor:"11", GitVersion:"v1.11.2", GitCommit:"bb9ffb1654d4a729bb4cec18ff088eacc153c239", GitTreeState:"clean", BuildDate:"2018-08-07T23:17:28Z", GoVersion:"go1.10.3", Compiler:"gc", Platform:"windows/amd64"}
> Server Version: version.Info{Major:"1", Minor:"11", GitVersion:"v1.11.2", GitCommit:"bb9ffb1654d4a729bb4cec18ff088eacc153c239", GitTreeState:"clean", BuildDate:"2018-08-07T23:08:19Z", GoVersion:"go1.10.3", Compiler:"gc", Platform:"linux/amd64"}

If you see `Unable to connect to the server: dial tcp [::1]:8080: connectex: No
connection could be made because the target machine actively refused it.` then
the kubeconfig is not being read properly or was not set up properly in the
previous labs.

If you prefer, the kubeconfig can be specified explicitly by using the
`--kubeconfig` flag for `kubectl` instead of `${env:KUBECONFIG}`. When running
the `kubelet` and `kube-proxy` below we'll set the `--kubeconfig` explicitly.
The `kubelet` uses the same kubeconfig as `kubectl`, while the `kube-proxy` uses
a slightly different kubeconfig.

### Configure CNI Networking

We'll use the Microsoft-provided `wincni` CNI plugin for networking on our
Windows worker nodes. Run these commands to fetch the CNI binary and put it into
place:

TODO: describe `wincni` in more detail. What does it do, how is it different
from other possible CNI plugins?

```
mkdir ${env:CNI_DIR}
git clone https://github.com/Microsoft/SDN.git ${env:K8S_DIR}\SDN
Copy-Item ${env:K8S_DIR}\SDN\Kubernetes\windows\cni\wincni.exe ${env:CNI_DIR}
```

TODO: build `wincni` from source instead of using the prebuilt binary? The
code is under `Kubernetes\wincni` in the repository, but it's unclear if/how
this code can be built (just a `go build` command?).

Now create the configuration file for "l2bridge" mode for the CNI plugin. This
config file is not well-documented, the contents are a modified version of what
the
[`start-kubelet.ps1`](https://github.com/Microsoft/SDN/blob/master/Kubernetes/windows/start-kubelet.ps1)
script generates.

```
$vethIp = (Get-NetAdapter | Where-Object Name -Like "vEthernet (*" |`
  Get-NetIPAddress -AddressFamily IPv4).IPAddress
$podCidr = "$([System.Text.Encoding]::ASCII.GetString((`
  Invoke-WebRequest -UseBasicParsing -H @{'Metadata-Flavor' = 'Google'} `
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/pod-cidr).Content))"

# For Windows nodes the pod gateway IP address is the .1 address in the pod
# CIDR for the host, but from inside containers it's the .2 address.
$podGateway = ${podCidr}.substring(0, ${podCidr}.lastIndexOf('.')) + '.1'
$podEndpointGateway = ${podCidr}.substring(0, ${podCidr}.lastIndexOf('.')) + '.2'

mkdir ${env:CNI_DIR}\config
$l2bridgeConf = "${env:CNI_DIR}\config\l2bridge.conf"
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
                "DestinationPrefix":  "VETH_IP/32",
                "NeedEncap":  true
            }
        }
    ]
}'.replace('POD_CIDR', ${podCidr}).`
replace('POD_ENDPOINT_GW', ${podEndpointGateway}).`
replace('VETH_IP', ${vethIp})
```

See [CNI config explanation](cni-config-explanation.md) for an explanation of
the fields in the CNI configuration. Note that if you need to edit the
configuration or correct a mistake you can open the file using `notepad++.exe`.

Note: see [CNI config
troubleshooting](troubleshooting/win-cni-config-troubleshooting.md) for some
informal notes I took while debugging this configuration.

TODO: instead of using these unsupported scripts that Microsoft ad-hoc released,
write scripts / executables that build on hcsshim instead and check them into
this repository.

### Configure the Host Networking Service

Continue with the Windows node network setup by running these commands to
configure the Host Networking Service (HNS). Note that your RDP session may be
interrupted when you invoke these commands.

```
$endpointName = "cbr0"
$vnicName = "vEthernet ($endpointName)"

Import-Module ${env:K8S_DIR}\SDN\Kubernetes\windows\hns.psm1

New-HNSNetwork -Type "L2Bridge" -AddressPrefix $podCidr -Gateway $podGateway `
  -Name ${env:KUBE_NETWORK} -Verbose
$hnsNetwork = Get-HnsNetwork | ? Type -EQ "L2Bridge"
$hnsEndpoint = New-HnsEndpoint -NetworkId $hnsNetwork.Id -Name $endpointName `
  -IPAddress $podEndpointGateway -Gateway "0.0.0.0" -Verbose
Attach-HnsHostEndpoint -EndpointID $hnsEndpoint.Id -CompartmentID 1 -Verbose
netsh interface ipv4 set interface "$vnicName" forwarding=enabled
Get-HNSPolicyList | Remove-HnsPolicyList
```

### Configure the Kubelet

Create the kubelet config file (which is different from the kubeconfig file...):

TODO: for authentication I've set anonymous to true instead of false since I
skipped RBAC setup (webhook) in an earlier lab. For authorization I've set mode
to AlwaysAllow instead of Webhook
(https://github.com/kubernetes/kubernetes/blob/cd78e999f9aade259c177c1698129311c83aa7d3/pkg/kubeapiserver/authorizer/modes/modes.go#L30).
Return to earlier lab and see if RBAC works there and here.

```
Set-Content ${env:KUBELET_CONFIG} `
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
tlsPrivateKeyFile: "K8S_DIR\HOSTNAME-key.pem"'.replace('K8S_DIR', `
${env:K8S_DIR}).replace('POD_CIDR', ${podCidr}).replace('HOSTNAME', `
$(hostname)).replace('\', '\\')
```

### Configure the Kubernetes Proxy

TODO: create a kube-proxy-config on the Windows node and move the kubeproxy
command-line flags from below to here. See corresponding section for Linux
worker nodes.

### Start the Worker Services

Finally, open a separate PowerShell window and start the `kubelet`:

```
& ${env:NODE_DIR}\kubelet.exe --hostname-override=$(hostname) --v=6 `
  --pod-infra-container-image=kubeletwin/pause --resolv-conf="" `
  --allow-privileged=true --config=${env:KUBELET_CONFIG} `
  --enable-debugging-handlers `
  --kubeconfig=${env:K8S_DIR}\$(hostname).kubeconfig --hairpin-mode=promiscuous-bridge `
  --image-pull-progress-deadline=20m --cgroups-per-qos=false `
  --enforce-node-allocatable="" --network-plugin=cni --cni-bin-dir="${env:CNI_DIR}" `
  --cni-conf-dir="${env:CNI_DIR}\config" --register-node=true
```

Note: to use Hyper-V isolation for the pods add
`--feature-gates=HyperVContainer=true`.

TODO: move the rest of the kubelet command-line flags into `${env:KUBELET_CONFIG}`.
See https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/ for
the flags that are supported.

Wait a few seconds, then open a new PowerShell window and then start kube-proxy:

```
& ${env:NODE_DIR}\kube-proxy.exe --v=4 --proxy-mode=kernelspace `
  --hostname-override=$(hostname) --kubeconfig=${env:K8S_DIR}\kube-proxy.kubeconfig `
  --cluster-cidr="10.200.0.0/16"
```

After a short delay the worker nodes should successfully join the cluster:

```
& ${env:K8S_DIR}\node\kubectl get nodes
```

> Output:

```
NAME            STATUS    ROLES     AGE       VERSION
worker-0        Ready     <none>    1h        v1.11.2
worker-1        Ready     <none>    1m        v1.11.2
worker-2        Ready     <none>    1m        v1.11.2
```

Note: see [Windows RBAC
troubleshooting](troubleshooting/windows-worker-cluster-join-RBAC.md) for notes
about issues I was having with getting Windows nodes to join when the apiserver
was started with Node,RBAC authorization.

## Verification

> The compute instances created in this tutorial will not have permission to
> complete this section. Run the following commands from the same machine used
> to create the compute instances.

List the registered Kubernetes nodes:

```
gcloud compute ssh controller-0 \
  --command "kubectl get nodes --kubeconfig admin.kubeconfig"
```

> output

```
NAME            STATUS    ROLES     AGE       VERSION
worker-0        Ready     <none>    1h        v1.11.2
worker-1        Ready     <none>    1m        v1.11.2
worker-2        Ready     <none>    1m        v1.11.2
```

TODO: in my Windows nodes the connection to the GCE metadata server was
disrupted by some of the configuration done here (or in an earlier or later
lab). This can be seen by looking at the serial console, which will show a
message like the one below. Figure out what steps causes this; check the hosts
file, check nslookup. A reboot doesn't help.

> 2018/08/20 22:07:05 GCEMetadataScripts: ERROR main.go:258: error connecting to
> metadata server, retrying in 15s, error: Get
> http://metadata.google.internal/computeMetadata/v1/instance/attributes/?recursive=true&alt=json&timeout_sec=10&last_etag=NONE:
> dial tcp: lookup metadata.google.internal: getaddrinfow: The requested name is
> valid, but no data of the requested type was found.

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
