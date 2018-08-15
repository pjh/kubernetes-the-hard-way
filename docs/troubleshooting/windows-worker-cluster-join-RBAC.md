journalctl --since="2018-08-10 19:35:51"

Now that wworker-2 is working with kubelet-config.yaml, try resetting authorization mode to Node,RBAC in controllers; make sure that healthz check works, make sure that Linux wworker-0 can join again, and then see if wworker-2 can join again! (wworker-1 probably will not be able to since it's not using kubelet-config.yaml yet).

Tried this, set verbosity on kube-apiserver on controllers to 6, found:

wworker-2 (Windows worker):
	rbac.go:118] RBAC DENY: user "system:kube-proxy" groups ["system:node-proxier" "system:authenticated"] cannot "list" resource "nodes" cluster-wide
	authorization.go:60] Forbidden: "/api/v1/nodes?fieldSelector=metadata.name%3Dwworker-2&limit=500&resourceVersion=0", Reason: ""
	wrap.go:42] GET /api/v1/nodes?fieldSelector=metadata.name%3Dwworker-2&limit=500&resourceVersion=0: (380.886µs) 403 [[kubelet.exe/v1.10.5 (windows/amd64) kubernetes/32ac1c9] 35.230.86.113:50196]
	...
	rbac.go:118] RBAC DENY: user "system:kube-proxy" groups ["system:node-proxier" "system:authenticated"] cannot "create" resource "nodes" cluster-wide
	authorization.go:60] Forbidden: "/api/v1/nodes", Reason: ""
	wrap.go:42] POST /api/v1/nodes: (823.195µs) 403 [[kubelet.exe/v1.10.5 (windows/amd64) kubernetes/32ac1c9] 35.230.86.113:50196]
		(NOTE: request comes from wworker-2's EXTERNAL address, for some reason.)

wworker-0 (Linux worker):
	handler.go:159] kube-aggregator: GET "/api/v1/nodes/wworker-0" satisfied by nonGoRestful
	pathrecorder.go:247] kube-aggregator: "/api/v1/nodes/wworker-0" satisfied by prefix /api/
	handler.go:149] kube-apiserver: GET "/api/v1/nodes/wworker-0" satisfied by gorestful with webservice /api/v1
	wrap.go:42] GET /api/v1/nodes/wworker-0: (3.993341ms) 200 [[kube-proxy/v1.10.5 (linux/amd64) kubernetes/32ac1c9] 104.196.230.18:57018]
	...
	trace.go:76] Trace[1965355180]: "Create /api/v1/namespaces/default/events" (started: 2018-08-10 18:39:16.712810291 +0000 UTC m=+816.620268760) (total time: 4.016581391s):
	wrap.go:42] POST /api/v1/namespaces/default/events: (4.019698329s) 201 [[kube-proxy/v1.10.5 (linux/amd64) kubernetes/32ac1c9] 104.196.230.18:57018]

Great, so I know that Windows is getting 403s (Forbidden), while Linux is getting 200 (OK) and 201 (Created). I know that Windows is using user "system:kube-proxy" which is part of groups "system:node-proxier" and "system:authenticated", but I don't know what user the Linux worker is using!

Windows kubelet log demonstrates the failure as:
  E0809 23:15:39.771028    3188 kubelet_node_status.go:106] Unable to register node "wworker-2" with API server: nodes is forbidden: User "system:kube-proxy" cannot create nodes at the cluster scope

A ha! Found that the --kubeconfig file passed to Windows (kube-proxy.kubeconfig) and Linux (/var/lib/kubelet/kubeconfig) kubelets is different, and appears to control the user that's used to connect to apiserver! (https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
	Linux has: user: system:node:wworker-0
	Windows has: user: system:kube-proxy     !!!
This is actually the only difference between kube-proxy.kubeconfig, which I was incorrectly using on Windows nodes, and $(hostname).kubeconfig, which I should be using.
	So when --authorization-mode=AlwaysAllow the user part of the context is not checked when worker kubelet connects to apiserver, but when --authorization-mode=Node,RBAC the user part of the context IS checked. Makes sense.

Finally, now I see same for Windows worker that I was seeing for Linux worker:
	handler.go:159] kube-aggregator: GET "/api/v1/nodes/wworker-2" satisfied by nonGoRestful
	pathrecorder.go:247] kube-aggregator: "/api/v1/nodes/wworker-2" satisfied by prefix /api/
	handler.go:149] kube-apiserver: GET "/api/v1/nodes/wworker-2" satisfied by gorestful with webservice /api/v1
	wrap.go:42] GET /api/v1/nodes/wworker-2: (4.04826ms) 200 [[kube-proxy.exe/v1.10.5 (windows/amd64) kubernetes/32ac1c9] 35.230.86.113:50235]
	...
	wrap.go:42] POST /api/v1/namespaces/default/events: (16.990439ms) 201 [[kube-proxy.exe/v1.10.5 (windows/amd64) kubernetes/32ac1c9] 35.230.86.113:50235]

Addendum: I think this approach might be "Node" authorization, not "RBAC".
https://kubernetes.io/docs/reference/access-authn-authz/node/ says: "In order to
be authorized by the Node authorizer, kubelets must use a credential that
identifies them as being in the system:nodes group, with a username of
system:node:<nodeName>.". "system:node:wworker-2" is the username that I'm now
setting in the Windows worker's kubelet kubeconfig.
