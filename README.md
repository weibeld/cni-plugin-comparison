# Kubernetes CNI plugin comparison

Scenario:

1. Create a Kubernetes cluster with default settings with kubeadm on GCP
2. Install a CNI plugin with default settings in the cluster
3. Check the network connectivities in the cluster

Compared CNI plugins:

- [Calico](#calico)
- [Flannel](#flannel)
- [Weave Net](#weave-net)
- [Cilium](#cilium)
- [Contiv](#contiv)
- [Kube-router](#kube-router)

## Summary table

| CNI plugin  | Works in cloud _(1)_ | Default Pod network CIDR  | Default Pod network CIDR defined in YAML _(2)_ | Works if `podCIDR` not allocated _(3)_ | Can override Pod network CIDR with kubeadm _(4)_ |
|-------------|:--------------:|---------------------:|:------------------------------:|:--------------------------:|:-------------------------------------------------:|
| Calico      | ❌ No          | 192.168.0.0/16       | ✅ Yes                         | ✅ Yes                     | ❌ No
| Flannel     | ✅ Yes         | 10.244.0.0/16        | ✅ Yes                         | ❌ No                      | ❌ No
| Weave Net   | ✅ Yes         | 10.32.0.0/12         | ❌ No                          | ✅ Yes                     | ❌ No
| Cilium      | ✅ Yes         | 10.217.0.0/16        | ❌ No                          | ✅ Yes                     | ✅ Yes
| Contiv      | ❌ No          | 10.1.0.0/16          | ✅ Yes                         | ✅ Yes                     | ❌ No
| kube-router | ❌ No          | -                    | ❌ No                          | ❌ No                      | ✅ Yes

Footnotes:

1. If no, it usually means that the inter-node Pod communication doesn't work. That means, no messages can be sent to a Pod on a _different_ node (to a Pod on the _same_ node works). This is because the CNI plugin probably assumes direct Layer 2 connectivity between nodes, which is not the case in the cloud.
2. If yes, the default Pod network CIDR can be customised by editing the deployment manifest of the CNI plugin.
3. If no, the CNI plugin Pods fail to start up if the `node.spec.podCIDR` field is not set, that is, if the controller manager didn't automatically allocate Pod subnet CIDRs to the nodes.
4. If yes, it's enough to define the Pod network CIDR by specifying it to kubeadm (which will pass it to the `--cluster-cidr` flag of [kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/); the CNI plugin will reuse the custom Pod subnet CIDRs assigned to each node. If no, this doesn't work, and the CNI plugin keeps using its default Pod network CIDR, even if a different CIDR was specified to kubeadm. More steps have to be taken to customise the Pod network CIDR, such as editing the CNI plugin deployment YAML file or binary.

## Calico

### Installation

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

See [documentation](https://docs.projectcalico.org/getting-started/kubernetes/quickstart).

### Connectivities

- Pod to itself: ✅
- Pod to Pod on same node: ✅
- Pod to own node: ✅
- Pod to Pod on different node: ❌
- Pod to different node: ❌
- Pod to Service IP address: ✅  if backend Pod is on same node as client Pod)
- Pod to Service DNS name: ❌
- DNS lookup: ❌

### Notes

- Does not require pre-existing Pod subnet CIDR allocation to nodes
  - If `node.spec.podCIDR` is set, Calico does not use it and uses a different Pod subnet CIDR for each node
- Creates a `calico-node` Pod on each node (host network)
- Additionally, creates a `calico-kube-controllers` Pod on one of the nodes (Pod network)
- Uses Pod network CIDR 192.168.0.0/16, by default
- Assigns each node a /24 subnet CIDR
- CoreDNS Pods are runnig on master node, but Pods can't do DNS lookups because Pod-to-Pod communication across nodes doesn't work

## Flannel

### Installation

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

See [documentation](https://github.com/coreos/flannel/#deploying-flannel-manually).

### First observations

Nodes become `Ready`. However, the `kube-flannel-ds` Pods on each node are in a run-crash loop. The `coredns` Pods are stuck in `ContainerCreating`.

The `kube-flannel-ds` Pods report the following error message in their log output:

```
Error registering network: failed to acquire lease: node "<NODE>" pod cidr not assigned
```

This is because there's no Pod subnet CIDR assigned to the nodes.

> Automatic assignments of Pod subnet CIDRs to the nodes happens only when you set the `networking.podSubnet` field in the `kubeadm init` configuration file or set the `--pod-network-cidr` flag of `kubeadm init`. 

You might try to fix this, you can manually assign a Pod subnet CIDR to each node:

```bash
kubectl edit node <NODE>
```

And change the `spec` field to contain the following:

```
spec:
  podCIDR: 200.200.0.0/24
```

Do this with a different CIDR for each node, for example, 200.200.0.0/24, 200.200.1.0/24, and 200.200.2.0/24.

Then, delete the individual `kube-flannel-ds` Pods to restart them (or wait unti they finish their `CrashLoopBackOff` and restart by themselves).

After that, the `kube-flannel-ds` Pods should be running correctly.

However, this doesn't make it work. The logs of the `kube-flannel-ds` Pods now show:

```
 Current network or subnet (10.244.0.0/16, 200.200.0.0/24) is not equal to previous one (0.0.0.0/0, 0.0.0.0/0), trying to recycle old iptables rules
```

And the Pods delete and recreate some iptables rules.

However, this doesn't make it work, as shown in the connectivities:

- Pod to itself: ✅
- Pod to Pod on same node: ❌
- Pod to own node: ✅
- Pod to Pod on different node: ❌
- Pod to different node: ❌
- Pod to Service IP address: ❌
- Pod to Service DNS name: ❌
- DNS lookup: ❌
- Pod on host network to own node: ✅
- Pod on host network to different node: ✅
- Pod on host network to Pod on same node: ✅
- Pod on host network to Pod on diferent node: ❌

### Observations when specifying an arbitrary Pod network CIDR

When using an arbitrary Pod network CIDR (e.g. 200.200.0.0/16) at cluster creation time, the following happens.

> The Pod network CIDR can be specified in the `networking.podSubnet` field in the `kubeadm init` config file or the `--pod-network-cidr` flag of `kubeadm init`. Kubernetes will then automatically assign a Pod subnet CIDR to each node.

All nodes are `Ready`, all Flannel Pods run, the `coredns` Pods get IP addreses and run, but are not ready.

When creating new workload Pods, they also get IP addresses and run, however, the connectivities are exactly as above when no Pod network CIDR is specified at all.

### Observations when specifying 10.244.0.0/16 as the Pod network CIDR

If you choose precisely 10.244.0.0/16 as the Pod network CIDR, the following happens.

All Flannel and `coredns` Pods get running and ready. All connectivities work:

- Pod to itself: ✅
- Pod to Pod on same node: ✅
- Pod to own node: ✅
- Pod to Pod on different node: ✅
- Pod to different node: ✅
- Pod to Service IP address: ✅
- Pod to Service DNS name: ✅
- DNS lookup: ✅
- Pod on host network to own node: ✅
- Pod on host network to different node: ✅
- Pod on host network to Pod on same node: ✅
- Pod on host network to Pod on diferent node: ✅

### Notes

- With Flannel, you **must** set the Pod network CIDR when you create the cluster and it **must** be 10.244.0.0/16
  - With all other configurations, Flannel will not work
- Creates a `kube-flannel` Pod on each node

## Weave Net

### Installation

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

See [documentation](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/).


### Connectivities

- Pod to itself: ✅
- Pod to Pod on same node: ✅
- Pod to own node: ✅
- Pod to Pod on different node: ✅
- Pod to different node: ✅
- Pod to Service IP address: ✅
- Pod to Service DNS name: ✅
- DNS lookup: ✅

### Notes

- Does not require pre-existing Pod subnet CIDR allocation to nodes
  - If you have pre-existing Pod subnet CIDR allocations, Weave Net does **not** use them but chooses its own Pod subnet CIDR (from a different Pod network CIDR than you specified) for each node
- Assigns a /24 Pod subnet CIDR to each node
- Runs a `weave-net` Pod on each node

## Cilium

### Installation

```bash
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/1.7.0/install/kubernetes/quick-install.yaml
```

See [documentation](https://cilium.readthedocs.io/en/stable/gettingstarted/k8s-install-default/).


### Connectivities

- Pod to itself: ✅
- Pod to Pod on same node: ✅
- Pod to own node: ✅
- Pod to Pod on different node: ✅
- Pod to different node: ✅
- Pod to Service IP address: ✅
- Pod to Service DNS name: ✅
- DNS lookup: ✅
- Pod on host network to own node: ✅
- Pod on host network to different node: ✅
- Pod on host network to Pod on same node: ✅
- Pod on host network to Pod on diferent node: ✅

### Notes

- Does not require pre-existing Pod subnet CIDR allocation to nodes
  - If you don't specify a Pod network CIDR, Cilium uses 10.217.0.0/16 by default
  - If you have pre-existing Pod subnet CIDR allocations, Cilium uses them as expected
- Runs a `cilium` Pod on each node
- Runs a `cilium-operator` Pod on one of the nodes (may be a worker node)

## Contiv

### Installation

```bash
kubectl apply -f https://raw.githubusercontent.com/contiv/vpp/master/k8s/contiv-vpp.yaml
```

See [documentation](https://github.com/contiv/vpp/blob/master/docs/setup/MANUAL_INSTALL.md#4-installing-the-contiv-vpp-cni-plugin).

### First observations

After deployment, the nodes remain `NotReady` and the `contiv-vswitch` Pods that Contiv deploys to each node, as well as the `coredns` Pods remain in `Pending`.

The reason is that Contiv requires [hugepages](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt) to run on the nodes.

To fix this issue, go to each node, and do the following:

```bash
sysctl -w vm.nr_hugepages=512
echo "vm.nr_hugepages=512" >> /etc/sysctl.conf
service kubelet restart
```

If you use Ansible, execute these commands on all hosts with:

```bash
ansible all -b -m shell -a 'sysctl -w vm.nr_hugepages=512; echo "vm.nr_hugepages=512" >> /etc/sysctl.conf; service kubelet restart'
```

After that, the nodes should become `Ready` and all the Pods should be scheduled and run.

However, the connectivities are like this:

- Pod to itself: ✅
- Pod to Pod on same node: ✅
- Pod to own node: ✅
- Pod to Pod on different node: ❌
- Pod to different node: ❌
- Pod to Service IP address: ✅ (if backend Pod is on same node as client Pod)
- Pod to Service DNS name: ❌
- DNS lookup: ❌
- Pod on host network to own node: ✅
- Pod on host network to different node: ✅
- Pod on host network to Pod on same node: ✅
- Pod on host network to Pod on diferent node: ❌

### Observations with a predefined arbitrary Pod network CIDR

When using an aribrary Pod network CIDR (e.g. 200.200.0.0/16), the connectivities are like above

### Observations with using 10.1.0.0/16 as the Pod network CIDR

According to the [documentation](https://github.com/contiv/vpp/blob/master/docs/setup/MANUAL_INSTALL.md#initializing-your-master), 10.1.0.0/16 must be used for the Pod network CIDR with kubeadm.

However, if using this Pod network CIDR, the connectivities are still like above.

### Notes

- Does not require pre-existing Pod subnet CIDR allocation to nodes
- If you specify a Pod network CIDR, Contiv does **not** use it. It uses the Pod network CIDR 10.1.0.0/16 instead.
  - This is hardcoded in the `contiv-vpp.yaml` file
- Deploys a `contiv-vswitch` Pod to each node as well as a `contiv-ksr`, `contiv-etcd`, and `contiv-crd` Pod to the master node
- Creates a single route for the whole Pod network on each node to a `vpp1` network interface
  - This network interface seem to be a FD.io/VPP (Vector Packet Processor) vSwitch, which is a fast, scalable layer 2-4 multi-platform network stack.
  - See [Contiv/VPP overview](https://fdio-vpp.readthedocs.io/en/latest/usecases/contiv/K8s_Overview.html)
  - The way the VPP switch connects nodes seems to be similar to Calico, that is, it doesn't work by default in a cloud environment

## Kube-router

### Installation

```bash
kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml
```

See [documentation](https://github.com/cloudnativelabs/kube-router/blob/master/docs/kubeadm.md#kube-router-providing-pod-networking-and-network-policy).

### First observations

Upon installation, the nodes become `Ready`, the `coredns` Pods are stuck in `ContainerCreating`, and the `kube-router` Pods keep crashing with a log output of:

```
Failed to get pod CIDR from node spec. kube-router relies on kube-controller-manager to allocate pod CIDR for the node or an annotation `kube-router.io/pod-cidr`
```

If you try to fix it by adding a `node.spec.podCIDR` field to all nodes (e.g. with `kubectl edit`), all Pods start running and new Pods also start up correctly and get IP addresses from the assigned Pod subnet CIDRs. However, the connectivities don't work:

- Pod to itself: ✅
- Pod to Pod on same node: ✅
- Pod to own node: ✅
- Pod to Pod on different node: ❌
- Pod to different node: ❌
- Pod to Service IP address: ✅ (if backend Pod is on same node as client Pod)
- Pod to Service DNS name: ❌
- DNS lookup: ❌
- Pod on host network to own node: ✅
- Pod on host network to different node: ✅
- Pod on host network to Pod on same node: ✅
- Pod on host network to Pod on diferent node: ❌

### Observations with an arbitrary predefined Pod network CIDR

All Pods start up and run correctly (however, `coredns` Pods are not ready). Pods in the Pod network get IP addresses from the allocated Pod subnet CIDRs.

However, the connectivities are exactly like above.

### Notes

- Creates a `kube-router` Pod on each node
- Requires pre-allocated Pod subnet CIDRs for each node (otherwise, fails to start up)
  - If there are pre-allocated Pod subnet CIDRs, it uses them
- Creates routes for each Pod subnet on each node. The routes for a Pod subnet on a different node go via a `tun` virtual network interface.
