## Configure docker to use a private registry

For this lab, we add a private registry to pull and search images.

*The docker registry is not actually used in the rest of the labs, so if you are running late you can skip this lab.*

**Objective:** Integrating a private registry is an important use case for customers. 

**Estimated time:** 10 minutes

## Edit Docker startup configuration

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

# Summary

This was a quick and simple lab to demonstrate how to configure Atomic host to use
a private docker registry. This will be a common pattern in enterprise
environments.

## [NEXT LAB](atomicDockerLVM.md)
