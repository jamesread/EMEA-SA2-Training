##Configure Kubernetes##

Before we get into this lab, lets review what we have done so far. 

1. Set up 3x RHEL Atomic hosts
2. Updated the hosts so they are all the same version
3. Configured etcd to store configuration information for Flannel
4. Started flannel on each host so that containers on one host can talk to
another.

Remember that Kubernetes is an orchestration layer, it has the responsibility
to manage many atomic hosts as a logical cluster and provision containers in
that cluster. Setting up the container instances, flannel networks and similar.

The kubernetes package provides several services

* kube-apiserver
* kube-scheduler
* kube-controller-manager
* kubelet, kube-proxy

These services are managed by systemd and the configuration resides in a
central location, `/etc/kubernetes`. We will break the services up between the
hosts.  The first host, *master*, will be the kubernetes master.  This host
will run kube-apiserver, kube-controller-manager, and kube-scheduler. In
addition, the master will also run _etcd_. The remaining hosts, or *nodes*, 
will run kubelet, proxy, cadvisor and docker.

###Prepare the hosts

* Backup the kubernetes configuration files on each system (master and nodes) before continuing.

```bash
for i in $(ls /etc/kubernetes/*); do cp $i{,.orig}; echo "Making a backup of $i"; done
```

**NOTE:** It is important that you replace any existing content in the config
files below with the new content provided.  You will find that the existing
content in the config files may not match what is provided below. This is expected.

Edit `/etc/kubernetes/config` to be the same on **all hosts**. For OpenStack
VMs we will be using the *private IP address* of the master host.  Make sure to 
substitute out the MASTER_PRIV_IP_ADDR placeholder below. Exit the containers
on each node when finished.

```
###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow_privileged=false"

# How the controller-manager, scheduler, and proxy find the apiserver
KUBE_MASTER="--master=http://MASTER_PRIV_IP_ADDR:8080"
```

#### Configure the kubernetes services on the master

Edit `/etc/kubernetes/apiserver` to appear as such.  Make sure to substitute
out the MASTER_PRIV_IP_ADDR placeholder below.  The portal_net IP addresses
need to be an IP address range not used anywhere else.  They do not need to be
routed.  They do not need to be assigned to anything.  It just must be an
unused block of addresses.  Kubernetes will assign "services" one of these
addresses.  But traffic to (or from) these addresses will NEVER leave a node.
It's actually the proxy on the local node that responds to these addresses.
This must be a different unused range than that assigned to flannel.  Flannel
specifies the addresses used by pods.  portal_net specifies the addresses used
by services.  But in both cases, no infrastructure changes are needed.  Just
pick an unused block of addresses.

```       
###
# kubernetes system config
#
# The following values are used to configure the kube-apiserver
#

# The address on the local server to listen to.
KUBE_API_ADDRESS="--address=0.0.0.0"
...
# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd_servers=http://MASTER_PRIV_IP_ADDR:4001"
...
```

Edit `/etc/kubernetes/controller-manager` to appear as such.  Substitute your
node IPs here in place of the MINION_PRIV_IP_{1,2} placeholder.

```
# Comma separated list of minions
KUBELET_ADDRESSES="--machines=MINION_PRIV_IP_1,MINION_PRIV_IP_2"
```

* Start the appropriate services on master:

```bash
for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; do 
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES 
done
```

####Configure the kubernetes services on the nodes

**NOTE:** Make these changes on each node.

***We need to configure and start the kubelet and proxy***

Edit `/etc/kubernetes/kubelet` to appear as below.  Make sure you substitute
kublet or node IP addresses appropriately. You have to make two changes
below.

```
# The address for the info server to serve on
KUBELET_ADDRESS="--address=0.0.0.0"

# this MUST match what you used in KUBELET_ADDRESSES on the controller manager
# unless you used what hostname -f shows in KUBELET_ADDRESSES.
KUBELET_HOSTNAME="--hostname_override=LOCAL_MINION_ETH0_ADDRESS"

# We are (mis)using KUBE_ETCD_SERVERS.  In a future release this will be KUBE_API_SERVERS.
KUBELET_API_SERVER="--api_servers=http://MASTER_PRIV_IP_ADDR:8080"

# Add your own!
KUBELET_ARGS=""
```

* edit `/etc/kubernetes/proxy` to appear as below.

```
# How the proxy find the apiserver
KUBE_PROXY_ARGS="--master=http://MASTER_PRIV_IP_ADDR:8080"
```

* Start the appropriate services on the nodes.

```bash
for SERVICES in kube-proxy kubelet docker; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done
```

*You should be finished!*

* Check to make sure the cluster can see the nodes from the master.

```
# kubectl get nodes
NAME                LABELS              STATUS
192.168.121.147     <none>              Ready
192.168.121.101     <none>              Ready
```

**The cluster should be running! Launch a test pod.**

## [NEXT LAB](deployApplication.md)
