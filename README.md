# EMEA SA2 Training

This is a set of labs that will probably take an afternoon, but if you do not 
finish them all within the afternoon you may continue at home. 

There are two options for running the labs, either on **OpenStack**, or on your 
**laptop**. While running them on your local laptop may seem appealing, **running them
on OpenStack will generally be less effort and will be faster**. 

## Preparation

### Choose your environment

1. A working KVM environment or
2. OpenStack environment

### Use correct networking

These series of labs use no less than 5 networks if deplying with OpenStack, or 
4 networks otherwise. It is obviously important to configure the correct
networks in the correct places, and have a good understanding of what you're
changing so as not to conflict. We'll mainly be working with 2x network ranges,
make sure you know what these are for you before you continue;

1. Flannel overlay network: xxx.xxx.xxx.xxx/16
2. Kubernetes Services network: xxx.xxx.xxx.xxx/24

## Order that the labs should be delivered:

1. [Deploy Atomic Hosts](deployAtomicHosts.md)
1. [Atomic LVM Storage](atomicDockerLVM.md)
1. [SPC Images / Containers](spcContainers.md)
1. [Configure Flannel](configFlannel.md)
1. [Configure Kubernetes](configKubernetes.md)
