# Explore the environment

What can you do?  What can't you do?  You may see a lot of "Command not found"
messages...  We'll explain how to get around that with the rhel-tools container
in a later lab. 

*If you are running late, you could skip this lab.*

**Objective**: Get a feel for atomic.

**Estimed time**: 10 minutes.

```
man tcpdump

git

tcpdump

sosreport
```
Why wouldn't we include these commands in the Atomic image?

# If you want to add something to Atomic Host, you must build a container

Let's try:

```
atomic install rhel7/rhel-tools
```

This will install the rhel-tools container, which can be used as the administrator's shell.

The container must be running before we can execute any commands, so let's start it first:

```
atomic run --name rhel-tools rhel7/rhel-tools
```

You'll see that the 'atomic' command has wrapped around Docker and has put you directly into the container by executing /bin/bash. You can have an explore of this container, but I recommend you exit from the console at this point and proceed.

Now let's try:

```
atomic run rhel7/rhel-tools man tcpdump
atomic run rhel7/rhel-tools tcpdump
```

You can also go into the rhel-tools container and explore its contents.

```
atomic run rhel7/rhel-tools /bin/sh
```

You might even want to create a shell script like the following on the Atomic Host as a helper script:

```
vi /usr/local/sbin/man
#!/bin/sh
atomic run rhel7/rhel-tools man $@

chmod +x /usr/local/sbin/man
```

This script makes using man pages transparent to the user (even though man pages are not installed on the Atomic Host, only in the rhel-tools container).
It could also be done with a bash alias.

```
man tcpdump
```

rhel-tools is a Super Privileged Container, which will be covered in the next presentation and lab.

# Summary

This lab was just to give you a tasted of RHEL Atomic, and what it feels like
to be on the shell. It reinforces the concept that thee is no `yum` or commands
that you may be familiar with on a standard RHEL distribution.

## [NEXT LAB](configDocker.md)
