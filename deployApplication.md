##Deploy an application##


* Create a file on master named `apache.json` that looks as such:

```json
{
    "apiVersion": "v1beta1",
    "desiredState": {
        "manifest": {
            "containers": [
                {
                    "image": "fedora/apache",
                    "name": "my-fedora-apache",
                    "ports": [
                        {
                            "containerPort": 80,
                            "protocol": "TCP"
                        }
                    ]
                }
            ],
            "id": "apache",
            "restartPolicy": {
                "always": {}
            },
            "version": "v1beta1",
            "volumes": null
        }
    },
    "id": "apache",
    "kind": "Pod",
    "labels": {
        "name": "apache"
    },
    "namespace": "default"
}
```

This JSON file is describing the attributes of the application environment. For example, it is giving it a "kind", "id", "name", "ports", and "image". Since the fedora/apache images doesn't exist in our environment yet, it will be pulled down automatically as part of the deployment process.

For more information about which options can go in the schema, check out the docs on the [kubernetes github page](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/docs).

* Deploy the fedora/apache image via the `apache.json` file.

```
kubectl create -f apache.json
```


* This command exits immediately, returning the value of the label, `apache`. You can monitor progress of the operations with these commands:
On the master (master) -

```
journalctl -f -l -xn -u kube-apiserver -u kube-scheduler
```

* On the Node (minion) -

```
journalctl -f -l -xn -u kubelet -u kube-proxy -u docker
```


* After the pod is deployed, you can also list the pod.  

```
# kubectl get pods
POD                 IP                  CONTAINER(S)        IMAGE(S)            HOST                LABELS              STATUS
apache              10.99.69.2          my-fedora-apache    fedora/apache       172.16.243.12/      name=apache         Running
```

The state might be 'Pending'. This indicates that docker is still attempting to download and launch the container.

* You can get even more information about the pod like this.

```
kubectl get pods --output=json apache
```

* Finally, on the Node (minion), check that the pod is available and running.

```
docker images
REPOSITORY TAG IMAGE ID CREATED VIRTUAL SIZE
kubernetes/pause latest 6c4579af347b 7 weeks ago 239.8 kB
fedora/apache latest 6927a389deb6 3 months ago 450.6 MB

docker ps -l
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
05c69c00ea48 fedora/apache:latest "/run-apache.sh" 2 minutes ago Up 2 minutes k8s--master.3f918229--apache.etcd--8cd6efe6_-_3a95_-_11e4_-_b618_-_5254005318cb--9bb78458
```

## Create a service to make the pod discoverable ##

Now that the pod is known to be running we need a way to find it.  Pods in kubernetes may launch on any minion and get an IP addresses from flannel.  So finding them is obviously not easy.  You don't want people to have to look up what minion the web server is on before they can find your web page!  Kubernetes solves this with a "service"  By default kubernetes will create an internal IP address for the service (from the portal_net range) which pods can use to find the service.  But we want the web server to be available outside the cluster.  So we need to tell kubernetes how traffic will arrive into the cluster destined for this webserver.  To do so we define a list of "publicIPs".  These need to be actual IP addresses assigned to actual minions.  In configurations like AWS or OpenStack where machines have both a public IP assigned somewhere "in the cloud" and the private IP assigned to the node, you must use the private IP.  This IP must be assigned to a minion and be visable on the minion via "ip addr." This is a list, you may list multiple nodes public IP.

* Create a service on the master by creating a `service.json` file

**NOTE:** You must use an actual IP address for the `publicIPs` value or the service will not run correctly on the minions

```json
{
    "apiVersion": "v1beta1",
    "containerPort": 80,
    "id": "frontend",
    "kind": "Service",
    "labels": {
        "name": "frontend"
    },
    "port": 80,
    "publicIPs": [
        "MINION_PRIV_IP_1"
    ],
    "selector": {
        "name": "apache"
    }
}
```

* Load the JSON file into the kubenetes system

```bash
kubectl create -f service.json
```

* Check that the service is loaded on the master

```bash
# kubectl get services
NAME                LABELS                                    SELECTOR            IP                  PORT
kubernetes-ro       component=apiserver,provider=kubernetes   <none>              10.254.207.162      80
frontend            name=frontend                             name=apache         10.254.195.231      80
kubernetes          component=apiserver,provider=kubernetes   <none>              10.254.8.30         443
```

* Check out how it by looking at the following commands on any minion

```bash
iptables -nvL -t nat
journalctl -b -l -u kube-proxy
```

* Finally, test that the container is actually working.

```
curl http://MINION_PRIV_IP_1/
Apache
```

* Now really test it.  If you are using OS1 you should be able to hit the web server from you web browser by going to the PUBLIC IP associated with the minion(s) you chose in your service.

```
firefox http://MINION_PUBLIC_IP_1/
```

* To delete the container.

```
kubectl delete pod apache
```

## Create a replication controller to control the pod ##

This should have the exact same definition of the pod as above, only now it is being controlled by a replication controller.  So if you delete the pod, or if the node disappears, the pod will be restarted elsewhere in the cluster!

* Create an `rc.json` file to describe the replication controller

```json
{
    "apiVersion": "v1beta1",
    "desiredState": {
        "podTemplate": {
            "desiredState": {
                "manifest": {
                    "containers": [
                        {
                            "image": "fedora/apache",
                            "name": "my-fedora-apache",
                            "ports": [
                                {
                                    "containerPort": 80,
                                    "protocol": "TCP"
                                }
                            ]
                        }
                    ],
                    "id": "apache",
                    "restartPolicy": {
                        "always": {}
                    },
                    "version": "v1beta1",
                    "volumes": null
                }
            },
            "labels": {
                "name": "apache"
            }
        },
        "replicaSelector": {
            "name": "apache"
        },
        "replicas": 1
    },
    "id": "apache-controller",
    "kind": "ReplicationController",
    "labels": {
        "name": "apache"
    }
}
```

* Load the JSON file on the master

```bash
kubectl create -f rc.json
```
* Check that the replication controller has started

```bash
# kubectl get rc
CONTROLLER          CONTAINER(S)        IMAGE(S)            SELECTOR            REPLICAS
apache-controller   my-fedora-apache    fedora/apache       name=apache         1
```

* The replication controller should have spawned a pod on a minion.  (This make take a short while, so STATUS may be Unknown at first)

```bash
# kubectl get pods
POD                                    IP                  CONTAINER(S)        IMAGE(S)            HOST                LABELS              STATUS
887b0ce0-ec3f-11e4-8c12-fa163e090f59   10.99.100.2         my-fedora-apache    fedora/apache       172.16.243.13/      name=apache         Running
```

Feel free to resize the replication controller and run multiple copies of apache.  Note that the kubernetes `publicIP` balances between ALL of the replicas!

```bash
# kubectl resize --replicas=3 replicationController apache-controller
resized

# kubectl get rc
CONTROLLER          CONTAINER(S)        IMAGE(S)            SELECTOR            REPLICAS
apache-controller   my-fedora-apache    fedora/apache       name=apache         3

# kubectl get pods
POD                                    IP                  CONTAINER(S)        IMAGE(S)            HOST                LABELS              STATUS
887b0ce0-ec3f-11e4-8c12-fa163e090f59   10.99.100.2         my-fedora-apache    fedora/apache       172.16.243.13/      name=apache         Running
e623955d-ec3f-11e4-8c12-fa163e090f59   10.99.69.3          my-fedora-apache    fedora/apache       172.16.243.12/      name=apache         Running
e6241e9c-ec3f-11e4-8c12-fa163e090f59   10.99.100.3         my-fedora-apache    fedora/apache       172.16.243.13/      name=apache         Running
```

I suggest you resize to 0 before you delete the replication controller.  Deleting a `replicationController` will leave the pods running.


## [NEXT LAB](summary.md)
