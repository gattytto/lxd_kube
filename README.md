## Kanonikal Kubersnaptes (LXD @ VM mode)

### Kubernetes in LXD with SnapD 

* os: latest ubuntu
* IPV6-only
* cri-o engine (using apt)
* calico+bird6 real and fake ranges
* vm-mode kubernetes nodes (using snap)
* asumes openvswitch vmbr0 bridge providing: 
  *  routing and automatic ipv6
  *  dhcp ipv4 + internet access
* asumes xfs/ext4 filesystem for vm's (overlayfs inside vm's limitations)

### NOTES
- Change profile_kube_base with ssh keys and timezone info accordingly, then add them to LXC

- Why VM's and not just containers?:

  - 1: kubernetes containers(from runtimes) bypass both kube/namespace/runtime and LXC memory boundaries, reporting host's memory (haskell: the impossible happened).

- overlayfs and containers([REF](https://www.eclipse.org/che/docs/che-7/installation-guide/installing-che-on-minikube/) for LXD CT mode setup):

  - VM-mode: if host filesystem is ZFS create a fixed size volume and use it's /dev/zdX endpoint to create a storage pool in LXD
  - CT-mode: container-mode LXC containers can share a zfs fixed size volume(using `mkfs.xfs -n ftype=1 /dev/zdX`) there's a prefixed folder in /etc/containers/storage/vm-name in crio configs patch that doesn't affect vm-mode setup behavior

### STEPS
* launch master
  ```sh
  lxc launch images:ubuntu/21.10/cloud -p default -p kube -p kube-master -p kube-vm --vm
  lxc list
  +-------------+---------+---------------------+--------------------------------------------------+-----------------+-----------+
  |    NAME     |  STATE  |        IPV4         |                      IPV6                        |      TYPE       | SNAPSHOTS |
  +-------------+---------+---------------------+--------------------------------------------------+-----------------+-----------+
  | fun-sunbeam | RUNNING | 10.0.1.158 (enp5s0) | 2001:----:----:----:----:----:----:d9bb (enp5s0) | VIRTUAL-MACHINE | 0         |
  +-------------+---------+---------------------+--------------------------------------------------+-----------------+-----------+
  ```
Spin-up process can be followed by:
```sh
lxc exec fun-sunbeam -- tail -f /var/log/syslog
lxc exec fun-sunbeam -- tail -f /var/log/cloud-init.log
```

* add calico networking (part 1)
  ```sh
  lxc exec fun-sunbeam -- sh -c 'export KUBECONFIG=/etc/kubernetes/admin.conf && snap install --classic kubectl && kubectl apply -f /root/kubeinlxd/calico.yaml'
  lxc exec fun-sunbeam -- sh -c 'export KUBECONFIG=/etc/kubernetes/admin.conf && kubectl apply -f https://docs.projectcalico.org/manifests/calicoctl.yaml'
  ```
  
  NOTE: before finishing calico setup worker node needs to be setup so calicoctl can be spin up
  
* launch slave
  ```sh
  lxc launch images:ubuntu/21.10/cloud -p default -p kube -p kube-slave -p kube-vm --vm
  lxc list
    +--------------+---------+---------------------+------------------------------------------------+-----------------+-----------+
  |     NAME     |  STATE  |        IPV4         |                      IPV6                        |      TYPE       | SNAPSHOTS |
  +--------------+---------+---------------------+--------------------------------------------------+-----------------+-----------+
  | beloved-oryx | RUNNING | 10.0.1.159 (enp5s0) | 2001:----:----:----:----:----:----:91f5 (enp5s0) | VIRTUAL-MACHINE | 0         |
  +--------------+---------+---------------------+--------------------------------------------------+-----------------+-----------+
  | fun-sunbeam  | RUNNING | 10.0.1.158 (enp5s0) | 2001:----:----:----:----:----:----:d9bb (enp5s0) | VIRTUAL-MACHINE | 0         |
  +--------------+---------+---------------------+--------------------------------------------------+-----------------+-----------+

  ```

* join the slave to the cluster
  ```sh
  lxc exec beloved-oryx -- sh -c "$(lxc exec fun-sunbeam -- sh -c 'kubeadm token create --print-join-command')"
  ```

* sign slave's CSR in master 
  ```sh
  lxc exec -t fun-sunbeam -- sh -c "export KUBECONFIG=/etc/kubernetes/admin.conf && kubectl get csr | grep Pending" | awk '{print $1}'| xargs -L1 lxc exec --env=KUBECONFIG=/etc/kubernetes/admin.conf fun-sunbeam -- /snap/bin/kubectl certificate approve
  ```

* create the IPPools
```sh
  lxc exec fun-sunbeam -- sh -c 'export KUBECONFIG=/etc/kubernetes/admin.conf && kubectl exec -ti -n kube-system calicoctl -- /calicoctl create -f - <<EOF
---
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: pods
spec:
  cidr: 1100:200::/104
  natOutgoing: true
  disabled: false
  nodeSelector: all()
---
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: service
spec:
  cidr: 1101:300:1:2::/112
  natOutgoing: true
  disabled: false
  nodeSelector: all()

EOF'
```

* Check everything went ok:
  ```verilog
  lxc exec --env=KUBECONFIG=/etc/kubernetes/admin.conf fun-sunbeam -- sh -c 'kubectl get all -Ao wide'
	NAMESPACE     NAME                                      READY   STATUS    RESTARTS      AGE   IP                                      NODE           NOMINATED NODE   READINESS GATES
	kube-system   pod/calico-node-j6d5f                     1/1     Running   0             87m   2001:----:----:----:----:----:----:d9bb   fun-sunbeam    <none>           <none>
	kube-system   pod/calico-node-mrk4r                     1/1     Running   0             64m   2001:----:----:----:----:----:----:91f5   beloved-oryx   <none>           <none>
	kube-system   pod/calicoctl                             1/1     Running   0             85m   2001:----:----:----:----:----:----:91f5   beloved-oryx   <none>           <none>
	kube-system   pod/coredns-64897985d-qgwzf               1/1     Running   0             88m   1100:200::3b:66c1                       fun-sunbeam    <none>           <none>
	kube-system   pod/coredns-64897985d-wvqk9               1/1     Running   0             88m   1100:200::3b:66c0                       fun-sunbeam    <none>           <none>
	kube-system   pod/etcd-fun-sunbeam                      1/1     Running   0             88m   2001:----:----:----:----:----:----:d9bb   fun-sunbeam    <none>           <none>
	kube-system   pod/kube-apiserver-fun-sunbeam            1/1     Running   0             88m   2001:----:----:----:----:----:----:d9bb   fun-sunbeam    <none>           <none>
	kube-system   pod/kube-controller-manager-fun-sunbeam   1/1     Running   2 (81m ago)   88m   2001:----:----:----:----:----:----:d9bb   fun-sunbeam    <none>           <none>
	kube-system   pod/kube-proxy-5cgmg                      1/1     Running   0             88m   2001:----:----:----:----:----:----:d9bb   fun-sunbeam    <none>           <none>
	kube-system   pod/kube-proxy-gdhw9                      1/1     Running   0             64m   2001:----:----:----:----:----:----:91f5   beloved-oryx   <none>           <none>
	kube-system   pod/kube-scheduler-fun-sunbeam            1/1     Running   2 (81m ago)   88m   2001:----:----:----:----:----:----:d9bb   fun-sunbeam    <none>           <none>

	NAMESPACE     NAME                 TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)                  AGE   SELECTOR
	default       service/kubernetes   ClusterIP   1101:300:1:2::1   <none>        443/TCP                  88m   <none>
	kube-system   service/kube-dns     ClusterIP   1101:300:1:2::a   <none>        53/UDP,53/TCP,9153/TCP   88m   k8s-app=kube-dns

	NAMESPACE     NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE   CONTAINERS    IMAGES                          SELECTOR
	kube-system   daemonset.apps/calico-node   2         2         2       2            2           kubernetes.io/os=linux   87m   calico-node   docker.io/calico/node:v3.21.2   k8s-app=calico-node
	kube-system   daemonset.apps/kube-proxy    2         2         2       2            2           kubernetes.io/os=linux   88m   kube-proxy    k8s.gcr.io/kube-proxy:v1.23.0   k8s-app=kube-proxy

	NAMESPACE     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                              SELECTOR
	kube-system   deployment.apps/coredns   2/2     2            2           88m   coredns      k8s.gcr.io/coredns/coredns:v1.8.6   k8s-app=kube-dns

	NAMESPACE     NAME                                DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                              SELECTOR
	kube-system   replicaset.apps/coredns-64897985d   2         2         2       88m   coredns      k8s.gcr.io/coredns/coredns:v1.8.6   k8s-app=kube-dns,pod-template-hash=64897985d
  ```

* Add the Youki runtime to the slave and create runtimeClass from the master
  ```sh
    lxc exec fun-sunbeam -- bash /root/kubeinlxd/youki.sh
    lxc exec beloved-oryx -- bash /root/kubeinlxd/youki.sh
  ```
