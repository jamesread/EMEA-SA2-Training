#**Configure Flannel**

Flannel is a major networking component of RHEL Atomic Host, which has one function - to allow containers running one host talk to containers running on another host. Without it, container networking could not span hosts. We are going to configure it in this lab. 

Check and explore the versions of software you have.  This should be the same on all nodes.

# Store the Flannel network configuration in etcd

Pick one of your Atomic hosts as the master for flannel. 

Take a moment to look at networking before flannel configuration, you should
see that is quite standard.

```
# ip a
```

`etcd` is a distributued system used for storing flannel configuration in
Atomic host. Before we start configuring Flannel we need to start its etcd as a
dependcy, start it now and enable it to come up after a reboot. 

Configure etcd to be reachable in your environment. Etcd runs on the master only 
in our setup and needs to be reachable by all nodes. The easy way is to allow etcd
to listen on all available interfaces. Restrict this as you see fit, in your 
environment. 

```
# vi  /etc/etcd/etcd.conf
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:4001"
```

 Start and enable etcd now.

```
# systemctl start etcd; systemctl enable etcd
# systemctl status etcd
```

We will configure Flannel by creating a `flannel-config.json` and then submit
that configuration to `etcd` for flannel to use. We will write the flannel
configuration to a temporary file. Create `/tmp/flannel-config.json` with this
content; 

```json
{
    "Network": "10.99.0.0/16",
    "SubnetLen": 24,
    "Backend": {
        "Type": "vxlan",
        "VNI": 1
     }
}
```

Looking at this configuration you may wonder way the subnet is specified twice
('/16' and 24 in 'SubnetLen'), but these are two different values. The subnet
specified in the 'Network' statement is for the entire overlay network, and the
second value in 'SubnetLen' is the subnet size (from the /16) assigned to each
host. Remember that a /16 network (65535 host addresses) is larger than a /24
network (254 host addresses). 

Submit the configuration to the etcd server. Use an IP address that the other nodes can reach the
master node with to upload the data. If this is an OpenStack VM you will need to look up the 
IP address on the OpenStack Horizon dashboard. Use the internal IP in this case.

```
# curl -L http://x.x.x.x:4001/v2/keys/coreos.com/network/config -XPUT --data-urlencode /tmp/flannel-config.json
```

On success, the `etcd` server will response with a JSON response. Here is an example of successful output:

```json
{"action":"set","node":{"key":"/coreos.com/network/config","value":"{\n    \"Network\": \"10.99.0.0/16\",\n    \"SubnetLen\": 24,\n    \"Backend\": {\n        \"Type\": \"vxlan\",\n        \"VNI\": 1\n     }\n}\n","modifiedIndex":3,"createdIndex":3}}
```

Flannel will use this configuration from `etcd` when it starts up.

You can verify the configuration exists by just doing a simple CURL without
setting any values;

```
# curl -L http://x.x.x.x:4001/v2/keys/coreos.com/network/config
```

# Configure Flanel startup options

The configuration that we stored in etcd in the last section is shared between
all Flannel hosts. We must now configure Flannel's startup options.

Configure Flannel to use the correct network interface. This is commonly `eth0`
but might be `ens0` or `em1`. Use `ip a` to list network interfaces. This
should not be necessary on most systems unless they have multiple network
interfaces.  In which case you will want to use the interface capable of
talking to other nodes in the cluster.

`/etc/sysconfig/flanneld`:

```
...
FLANNEL_ETCD="http://x.x.x.x:4001"
...
FLANNEL_OPTIONS="eth0"
```

* Enable the flanneld service and reboot.

```
# systemctl enable flanneld
# systemctl reboot
```

Flannel is now a dependency for Docker - the `docker0` and `flannel.1` network
interfaces must be on the same subnet otherwise docker will fail
to start. If Docker fails to load, or the flannel IP is not set correctly,
reboot the system. 

When the system comes back up check the interfaces on the host now. Notice
there is now a flannel.1 interface.

```
# ip a
...<snip>...
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether fa:16:3e:32:ba:98 brd ff:ff:ff:ff:ff:ff
    inet 172.16.36.47/24 brd 172.16.36.255 scope global dynamic eth0
       valid_lft 182sec preferred_lft 182sec
    inet6 fe80::f816:3eff:fe32:ba98/64 scope link 
       valid_lft forever preferred_lft forever
3: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN 
    link/ether c2:13:e3:a3:ae:3e brd ff:ff:ff:ff:ff:ff
    inet 10.99.25.0/16 scope global flannel.1
       valid_lft forever preferred_lft forever
    inet6 fe80::c013:e3ff:fea3:ae3e/64 scope link 
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN 
    link/ether 56:84:7a:fe:97:99 brd ff:ff:ff:ff:ff:ff
    inet 10.99.25.1/24 scope global docker0
       valid_lft forever preferred_lft forever

```

You see in the above example that the 10.99.x.x interface is shared between
Docker and Flannel. 

Now that master is configured, lets configure the other nodes called "minions" (minion{1,2}).

# Configure the nodes

The shared configuration that we set in `etcd` will also be read by the
minions. Lets check that we can read it;

* Use curl to check firewall settings from each minion to the master.  We need
to ensure connectivity to the etcd service.  You may want to set up your
`/etc/hosts` file for name resolution here.  If there are any issues, just
fall back to IP addresses for now. 

**NOTE:** For OpenStack nodes use the *private IP address* of the master, not
the public IP address that we used before. 

```
# curl -L http://x.x.x.x:4001/v2/keys/coreos.com/network/config
```

If it looks like you get a valid JSON configuration, you are ready to continue
and configure flannel like the master, but use the IP address from the `curl`
command above;

`/etc/sysconfig/flanneld` contents:

```
...
FLANNEL_ETCD="http://x.x.x.x:4001"
...
FLANNEL_OPTIONS="eth0"
```

* Restart flanneld on both of the minions.

```
# systemctl restart flanneld
# systemctl enable flanneld
```

Check the new interface on both of the minions has a subnet that is *within*
the range you configured (eg: 10.99.x.x);

```
# ip a l flannel.1
```

Note that if you are using a /16 overlay network and a /24 host network as per
the example, then you may have a master on 10.99.25.0/24 and a minion on
10.99.30.0/24 - both of these networks are within the 10.99.0.0/16 network.

From any node in the cluster, check the cluster members by issuing a query to
etcd via curl.  You should see that three servers have consumed subnets.  You
can associate those subnets to each server by the MAC address that is listed in 
the output.

## List all subnets in the flannel configuration

```
# curl -L http://x.x.x.x:4001/v2/keys/coreos.com/network/subnets | python -mjson.tool
```

The curl command gets all the subnets in the flannel network, you should see
your 3 nodes listed. By piping the output to `python -mjson.tool` Python will
format the JSON with line breaks so it is a little easier to read.

## Review the flannel configuration on a host

From all nodes, review the `/run/flannel/subnet.env` file.  This file was
 generated automatically by flannel.


```
# cat /run/flannel/subnet.env
```

You should see output that contains your `FLANNEL_SUBNET`. 

* Docker will fail to load if the docker and flannel network interfaces are not setup correctly. Again it is possible to fix this by hand, but rebooting is easier.

```
# systemctl reboot
```

* A functioning configuration should look like the following; notice the docker0 and flannel.1 interfaces.


```
# ip a
1: lo:  mtu 65536 qdisc noqueue state UNKNOWN group default
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
inet6 ::1/128 scope host
valid_lft forever preferred_lft forever

2: eth0:  mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
link/ether 52:54:00:15:9f:89 brd ff:ff:ff:ff:ff:ff
inet 192.168.121.166/24 brd 192.168.121.255 scope global dynamic eth0
valid_lft 3349sec preferred_lft 3349sec
inet6 fe80::5054:ff:fe15:9f89/64 scope link
valid_lft forever preferred_lft forever

3: flannel.1:  mtu 1450 qdisc noqueue state UNKNOWN group default
link/ether 82:73:b8:b2:2b:fe brd ff:ff:ff:ff:ff:ff
inet 18.0.81.0/16 scope global flannel.1
valid_lft forever preferred_lft forever
inet6 fe80::8073:b8ff:feb2:2bfe/64 scope link
valid_lft forever preferred_lft forever

4: docker0:  mtu 1500 qdisc noqueue state DOWN group default
link/ether 56:84:7a:fe:97:99 brd ff:ff:ff:ff:ff:ff
inet 18.0.81.1/24 scope global docker0
valid_lft forever preferred_lft forever
```

Do not move forward until all three nodes have the docker and flannel
interfaces on the same subnet.

At this point the flannel cluster is set up and we can test it. We have etcd
running on the master node and flannel / Docker running on minion{1,2} minions.
Next steps are for testing cross-host container communication which will
confirm that Docker and flannel are configured properly.

## Test the flannel configuration

In this section we are going to spawn a container on two different hosts, and
ping between the containers using the flannel network. Unfortunately the rhel7
container image does not include `ifconfig` or `ip` to easily find the IP
address of a container, but the **fedora:20** image and **rhel6:latest* do have these
commands. Lets spawn one of each container. 

* Issue the following on minion1.

```
# atomic install registry.access.redhat.com/rhel6
# atomic run --name=rhel6 rhel6
# atomic run rhel6 /bin/bash
```

* This will place you inside the container. Check the IP address.

```
# ip a l eth0
5: eth0:  mtu 1450 qdisc noqueue state UP group default
link/ether 02:42:0a:00:51:02 brd ff:ff:ff:ff:ff:ff
inet 10.99.25.2/24 scope global eth0
valid_lft forever preferred_lft forever
inet6 fe80::42:aff:fe00:5102/64 scope link
valid_lft forever preferred_lft forever
```

You can see here that the IP address is on the flannel network.

* Issue the following commands on minion2:

```

# atomic install registry.access.redhat.com/rhel6
# atomic run --name=rhel6 rhel6
# atomic run rhel6 /bin/bash

# ip a l eth0
5: eth0:  mtu 1450 qdisc noqueue state UP group default
link/ether 02:42:0a:00:45:02 brd ff:ff:ff:ff:ff:ff
inet 10.99.30.2/24 scope global eth0
valid_lft forever preferred_lft forever
inet6 fe80::42:aff:fe00:4502/64 scope link
valid_lft forever preferred_lft forever
```

* Now, from the container running on minion2, ping the container running on minion1:

```
# ping -c 3 10.99.25.2 
PING 10.99.25.2 (18.0.81.2) 56(84) bytes of data.
64 bytes from 10.99.25.2: icmp_seq=2 ttl=62 time=2.93 ms
64 bytes from 10.99.25.2: icmp_seq=3 ttl=62 time=0.376 ms
64 bytes from 10.99.25.2: icmp_seq=4 ttl=62 time=0.306 ms
```

You should have received a reply. That is it. flannel is set up on the two
minions and you have cross host communication. `etcd` is set up on the master
node. Do not move forward until you can ping from container to container on
different hosts.

Next step is to overlay the cluster with kubernetes.

Exit the containers on each node when finished.
```
# atomic stop rhel6
```

##**Troubleshooting**

### Restarting services

Restart services in this order:

1. etcd
1. flanneld
1. docker

### Networking

Flannel configures an overlay network that docker uses. `ip a` must show docker and flannel on the same network.

Flannel has file `/usr/lib/systemd/system/docker.service.d/flannel.conf` which sources `/run/flannel/docker`, generated from the `flannel-config.json` file. etcd stores the flannel configuration for the Master. Flannel runs on each node host (minion) to setup a unique class-C container network.

## [NEXT LAB](configKubernetes.md)
