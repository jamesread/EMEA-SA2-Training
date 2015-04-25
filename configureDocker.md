## Configure docker to use a private registry
Integrating a private registry is an important use case for customers. For this lab, we add a private registry to pull and search images.

* Edit the `/etc/sysconfig/docker` file and restart docker. You will need the following two lines;

```
ADD_REGISTRY='--add-registry my.private.registry.fqdn'
INSECURE_REGISTRY='--insecure-registry my.private.registry.fqdn'
```

**NOTE:** We use the INSECURE_REGISTRY statement because if the private
registry is not configured with a CA-signed SSL
certificate `docker pull ...` will fail with a message about an insecure
registry. 

* Restart docker so that the service updates its configuration.

```
systemctl restart docker
```

This concludes the deploying Atomic lab.

