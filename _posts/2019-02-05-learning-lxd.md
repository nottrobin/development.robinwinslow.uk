---
title: How to use Linux Containers with LXD, and why you might want to
date: 2019-02-05 00:00:00 Z
description: Getting up and running with LXD containers - Quick usage; basic networking;
  sharing folders with write access
image_url: https://upload.wikimedia.org/wikipedia/commons/4/40/Linux_Containers_logo.png
layout: post
---

*I initially [drafted this](https://github.com/nottrobin/robinwinslow.uk/blob/66be16a8281206c7fd41d7b403be2895f4b49c24/_posts/learning-lxd.md) nearly 3 years ago. I didn't publish it then (for no good reason), so I'm publishing it now. I have gone through the article and made sure it still works with the latest LXD at the time of publishing. I also updated it to use `bionic` instead of `xenial`, and to cover [the snap version of LXD][lxd-snap] as well as [the deb][lxd-deb].*

---

[Linux Containers (LXC)](https://linuxcontainers.org/) effectively spin up mini virtual operating systems inside Linux, except that they're much more light-weight, and therefore much quicker, than true [virtual machines][VMs].

[LXD][] is a new system-wide daemon from Canonical's [Stéphane Graber](https://twitter.com/stgraber) which improves performance & security, and also offers an improve interface for the `lxc` command.

You may want to skip straight to [Installing LXD](#installing-lxd) or [How to use LXD](#how-to-use-lxd).

# tl;dr summary

Once you've [installed LXD][install], create a new container and open a bash terminal with:

``` bash
lxc launch ubuntu:bionic caged-beaver   # Download 18.04 Bionic Beaver image and start the container
lxc exec caged-beaver -- bash           # Run and attach to a bash terminal in the container
```

# Why are containers so useful?

As containers are like lighter virtual machines, you can use them for very much the same things, but do them *much* faster:

- Install & run things in a different version of Linux than is installed locally
- Run a software project within a predictable context for development (e.g. many
  people currently use [Vagrant][] for this)
- Test that things still work in a clean install of your operating system

Also, as containers are phenomenally faster (they can take less than one second to start) and use much less resources, they also bring with them new possibilities:

- Run many containers side-by-side, to test out large networks of interconnected services
- Run each individual process in its own container with encapsulated
  dependencies (rather than whole software projects)

## Initialising LXD

LXD should be available by default in Ubuntu 16.04 Xenial and newer. You can see what version you have with `lxc --version` - if it's v2 or newer, these instructions should work for you.

If you don't have `lxc` v2 or newer, refer to [the official installation guide](https://linuxcontainers.org/lxd/getting-started-cli/).

Now initialise LXD, with a `dir` backend and an LXD bridge:

``` bash
$ sudo lxd init
Name of the storage backend to use (dir or zfs): dir
Would you like LXD to be available over the network (yes/no)? no
Do you want to configure the LXD bridge (yes/no)? yes
```

Now the "Package configuration" will open and ask you to configure LXD. If you're unsure about any of the questions, just select the default option by pressing `<enter>`.

Read the [official installation guide][install].

# How to use LXD

When I first drafted this post, the latest version of LXD was `2.0.0`. At the time of publishing, it's now [`3.9`](https://discuss.linuxcontainers.org/t/lxd-3-9-has-been-released/3732) - however I believe all the commands mentioned below still work.

## Quickly start a container

To download and start a container running [Ubuntu 18.04 Bionic Beaver][bionic], simply run:

``` bash
lxc launch ubuntu:bionic caged-beaver  # "caged-beaver" could be any name you choose
```

It will take a while the first time as it has to download the OS image, but to create subsequent containers from the same image should only take a matter of seconds.

Once it's finished, check it's running:

``` bash
$ lxc list
+--------------+---------+----------------------+----------------+------------+-----------+
|     NAME     |  STATE  |         IPV4         |      IPV6      |    TYPE    | SNAPSHOTS |
+--------------+---------+----------------------+----------------+------------+-----------+
| caged-beaver | RUNNING | 10.94.108.174 (eth0) | a:a:a:a (eth0) | PERSISTENT |           |
+--------------+---------+----------------------+----------------+------------+-----------+
```

## Running commands

To run a command in our new container, use the `lxc exec` command. The command will run as the `root` user, in the `/root` directory inside the container.

You can run any command that you would normally run in a Linux system, but the most immediately useful command to run is `bash`, which will drop you into a bash shell inside the container:

``` bash
$ lxc exec caged-beaver -- bash
root@caged-beaver:~# echo "hello from inside the container!"
hello from inside the container!
root@caged-beaver:~#
```

## Networking

If you [initialised LXD earlier](#initialising-lxd) with a network bridge, your container should already have an IP address which you can connect to directly:

``` bash
$ lxc list caged-beaver --format=json | jq -r '.[0]["state"]["network"]["eth0"]["addresses"][0]["address"]'  # Find the IP address
10.95.60.183
$ ping 10.95.60.183  # Check we can see that IP address
64 bytes from 10.95.60.183: icmp_seq=1 ttl=64 time=0.134 ms
```

You can use this IP address for accessing any servers that you run in your container. If you need to use different network settings, simply run `lxd init` again.

## Sharing folders

You can share folders with your container as "devices". The syntax is:

``` bash
lxc config device add {container-name} {device-name} disk source={full-path-to-folder} path={path-inside-container}
```

So if we want to create a `share` directory and share it with the container at `/media/share`:

``` bash
mkdir share  # Create a new "share" directory
touch share/hello  # Create a "hello" file inside the directory
lxc config device add caged-beaver shareddir disk source=`pwd`/share path=/media/share  # Share it with the container
```

Now let's check the device is set up and that we can see it inside the container:

``` bash
$ lxc config device list caged-beaver  # List all devices in the container
shareddir
$ lxc exec caged-beaver -- ls -l /media/share
total 0
-rw-rw-r-- 1 nobody nogroup 0 May 16 16:55 hello
```

### Gaining write access to shared folders

You'll notice that the file inside `/media/share` in the container is owned by `nobody:nogroup`. This is true of the directory as well, and means that root can't actually edit or create files inside the shared directory:

``` bash
$ lxc exec caged-beaver -- touch /media/share/hello
touch: cannot touch '/media/share/hello': Permission denied
```

This is because the permissions inside the container are exactly the same as in the host system - as in the directory is still owned by *your user*. The container runs as an unprivileged user and so doesn't have access to your user's things.

To fix this, we can extend the permissions on the share directory to give the container's user access to it. Let's look at the permissions it has currently:

``` bash
$ getfacl share
# file: share
# owner: {your-user}
# group: {your-user}
user::rwx
group::rwx
other::r-x
```

Now we need to add permission for the LXD user IDs to access this folder. These IDs are usually `165536` (`root`) and `166536` (`ubuntu`) for the [deb version][lxd-deb], and `1000000` (`root`) and `1001000` (`ubuntu`) for the [snap version][lxd-snap]. These UIDs can be checked [as explained here](https://gist.github.com/nottrobin/032c7907c506f99e5fb31255f2f6ff2a).

Now we'll simply give all four UIDs for LXD access to our local directory with `setfacl`:

``` bash
$ setfacl -Rm user:lxd:rwx,default:user:lxd:rwx,user:165536:rwx,default:user:165536:rwx,user:166536:rwx,default:user:166536:rwx,user:1000000:rwx,default:user:1000000:rwx,user:1001000:rwx,default:user:1001000:rwx share
$ getfacl share  # Check permissions again
# file: share
# owner: {your-user}
# group: {your-user}
user::rwx
user:lxd:rwx
user:165536:rwx
user:166536:rwx
user:1000000:rwx
user:1001000:rwx
group::r-x
mask::rwx
other::r-x
default:user::rwx
default:user:lxd:rwx
default:user:165536:rwx
default:user:166536:rwx
default:user:1000000:rwx
default:user:1001000:rwx
default:group::r-x
default:mask::rwx
default:other::r-x
```

You should now be able to create and edit files in the shared folder from within the container:

``` bash
$ lxc exec caged-beaver -- touch /media/share/hello-again
$ ls share
hello  hello-again
```

## Privileged containers

We can also make containers "privileged", which sets the root user inside the container to be the same as the root user on the host system.

This would have been a way to solve the shared folders problem above, but it is unsafe (and in that case unnecessary), as it means that programs inside the container can escape into root on the host system in some circumstances.

First, note that the `rootfs` directory in your container is currently owned by an unprivileged user:

``` bash
$ ls -ld /var/lib/lxd/containers/caged-beaver/rootfs  # For the Deb version
drwxr-xr-x 22 165536 165536 4096 May 17 17:18 /var/lib/lxd/containers/caged-beaver/rootfs
# OR
$ sudo ls -ld /var/snap/lxd/common/mntns/var/snap/lxd/common/lxd/storage-pools/default/containers/caged-beaver/rootfs  # For the snap version
drwxr-xr-x 22 1000000 1000000 22 Jan 31 15:51 /var/snap/lxd/common/mntns/var/snap/lxd/common/lxd/storage-pools/default/containers/caged-beaver/rootfs
```

To make the container privileged, and then restart it:

``` bash
lxc config set caged-beaver security.privileged true
lxc restart caged-beaver
```

And note that the `rootfs` owner is now `root` in the host:

``` bash
$ ls -ld /var/lib/lxd/containers/caged-beaver/rootfs  # For the Deb version
drwxr-xr-x 22 root root 4096 May 17 17:18 /var/lib/lxd/containers/caged-beaver/rootfs
# OR
$ sudo ls -ld /var/snap/lxd/common/mntns/var/snap/lxd/common/lxd/storage-pools/default/containers/caged-beaver/rootfs  # For the snap version
drwxr-xr-x 22 root root 22 Feb  5 15:12 /var/snap/lxd/common/mntns/var/snap/lxd/common/lxd/storage-pools/default/containers/caged-beaver/rootfs
```

This means that the root user inside the container can now manipulate files on the host system as the root user.

Read more about [unprivileged vs privileged containers][stgraber-unpriv].

## Launching other images

In the command we ran earlier:

``` bash
lxc launch ubuntu:bionic caged-beaver
```

"ubuntu" is the name of a remote image list and "bionic" is the alias of an image from that list.

To find other images, there are several list commands:

``` bash
lxc remote list               # See what other remotes are available
lxc image list ubuntu:        # List all images in the "ubuntu" remote
lxc image alias list ubuntu:  # List every alias in the "ubuntu" remote
lxc image alias list images:  # List every alias in the "images" remote
```

From the last command, we can see that there's an image available in the "images" remote with the alias "alpine/edge/amd64", so we could choose to launch an alpine container in the same way:

``` bash
lxc launch images:alpine/edge/amd64 caged-mountain  # Again "caged-mountain", could be any name you choose
```

[LXD]: https://linuxcontainers.org/lxd/ "Linux Containers: What's LXD?"
[VMs]: https://en.wikipedia.org/wiki/Virtual_machine "Virtual machines"
[Vagrant]: https://www.vagrantup.com/ "Vagrant homepage"
[install]: https://linuxcontainers.org/lxd/getting-started-cli/ "Installing LXD and the command line tool"
[bionic]: https://wiki.ubuntu.com/BionicBeaver/ReleaseNotes "Ubuntu wiki: Bionic Beaver 18.04 release notes"
[install other remotes]: https://linuxcontainers.org/lxd/getting-started-cli/#importing-some-images "LXD official documentation: Importing some images"
[stgraber-unpriv]: https://www.stgraber.org/2014/01/17/lxc-1-0-unprivileged-containers/ "LXC 1.0: Unprivileged containers [7/10] - Introduction to unprivileged containers"
[lxd-snap]: https://snapcraft.io/lxd "Snapcraft: The LXD snap"
[lxd-deb]: https://packages.ubuntu.com/source/bionic/lxd "Ubuntu packages: lxd source package"
