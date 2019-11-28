---
title: meta-cloonix
date: 2019-10-05T12:00:00-05:00
draft: false
toc: true
featured_image: 'media/yocto-way0.jpg'
---

## Introduction

One of the features that I most like when using [Yocto Project](https://www.yoctoproject.org/) is the possibility to use [qemu](https://www.qemu.org/) and test my custom image very early without the needed of any hardware board.

All of that is integrated when using Yocto Project. See more details about how to use in [Using the Quick EMUlator (QEMU)](https://www.yoctoproject.org/docs/2.7.1/dev-manual/dev-manual.html#dev-manual-qemu).

But running one qemu instance is not enough when I need to test how an application behaved in a network environment.

You can ending up like these commands:

{{< gist joaohf c77f1046e4e3a1c1b911501333f5de90 >}}

A better way must exist to run that. So, why not a network emulator system?

I know that there are many network emulator/simulator systems. I've been reading this website  [Open-Source Routing and Network Simulation](http://www.brianlinkletter.com/) which has nice reviews about many network simulators.

### What is cloonix ?

I was looking for some specific requirements when searching about the subject:

* integrated network simulation
* uses qemu
* able to run any Linux image
* a nice way to describe my network setup
* integration with [DPDK](https://www.dpdk.org/) and [Open vSwitch](https://www.openvswitch.org/)

I've found [cloonix](https://clownix.net/). The source code is under AGPLv3 and free available at https://github.com/clownix/cloonix

cloonix seems to be very well designed and implemented, so I've gave a try reading and following the official documentation steps. After some exploration, it works very well. But how to use cloonix with my own Yocto images?

That's why I spend a bit more time to better understand what were the needed requirements to build a Yocto image suitable to run with cloonix.

I recommend to take a look over cloonix website ~~and~~ discovery what else it can do.

## Creating cloonix images

While the cloonix documentation has a very nice way to [create additional VMs](https://clownix.net/doc_stored/build-03-04/html/doc/vm_create_from_iso.html) and also providing already [ready-to-use VMs](http://clownix.net/downloads/qcow2_virtual_machines/); I would like an easy way to use Yocto images with cloonix.

So, I've dug in cloonix source code and documentation to discovery how it works and how a Yocto image could be created.

I figured out that there are two main points:

1. use the correct linux kernel append parameters
2. configure the file '/etc/inittab' (or systemd service) to pass the correct parameters to _getty_ (or _agetty_ when systemd)

Using the correct parameter, cloonix can access the guest using the _hvc0_ console and run two programs:

* _cloonix\_agent_: an agent which receives command from cloonix, running in host
* _dropbear\_cloonix\_ssh_: a ssh daemon over cloonix_agent channel. So we can access the guest using ssh without a network connection

## meta-cloonix

_meta-cloonix_ is a Yocto/Openembedded layer-compatible which allows you to create a image and use it with cloonix network simulator.

The source code layer can be found here: https://github.com/joaohf/meta-cloonix and so far this has a image and additional bbappend files.

But in a long-term this layer could build and install the host tools to talk with cloonix instance.

## Using meta-cloonix layer to create images

Clone `meta-cloonix` from https://www.github.com/joaohf/meta-cloonix repository and enable it in the file `conf/bblayers.conf`. See how to do this in [Enabling Your Layer](https://www.yoctoproject.org/docs/latest/dev-manual/dev-manual.html#enabling-your-layer).

In the file `conf/local.conf` we need to add two additional configurations:

* Enable the cloonix DISTRO_FEATURE:
{{< highlight bash >}}
DISTRO_FEATURES_append = " cloonix"
{{< / highlight >}}

* Use the cloonix specific WKS_FILE:
{{< highlight bash >}}
WKS_FILE = "cloonix-qemux86-directdisk.wks"
{{< / highlight >}}

These two configurations will enable an image which been created with:

* correct linux kernel append flags
* special configuration to enable inittab spawing a hvc0 console

meta-cloonix provides an image, based com _core-image_minimal_, which we can use to build a basic image to test the Cloonix environment:

{{< highlight bash >}}
bitbake cloonix-image-minimal
{{< / highlight >}}

After the build, copy the image file to CLOONIX_BULK directory:

{{< highlight bash >}}
cp tmp/deploy/images/qemux86/cloonix-image-minimal-qemux86.wic.qcow2 $HOME/cloonix_data/bulk
{{< / highlight >}}

Right now we created a image suitable to use with cloonix with Yocto. Remember that you are free to install any kind of extra software and also enable other layers.

## Building a network using cloonix

Now, we will work only with Cloonix command and graphic interface to build a small network within some nodes sending packets and a protocol sniffer, allowing seen what is going on in the network.

In this post we will build a script which does all the needed configuration.

The diagram below shows the network that I want to create:

```
          +---+
          |snf|
          +-+-+
            |
+---+       |       +---+
|kvm+-------+-------+kvm|
+---+      lan      +---+
```

See more about the cloonix objects used in this example here [cloonix objects](https://clownix.net/doc_stored/build-03-04/html/doc/objects.html).

So far, our network has:

* two _kvm_ guests (using the cloonix-image-minimal)
* a _lan_ to connect these two kvm
* a _snf_ (sniffer) listening all the traffic between the hosts

Below, the following cloonix script (I've borrow some ideas from [clownix/freeswitch](https://github.com/clownix/freeswitch)) is used to create all the objects:

{{< highlight bash "linenos=inline">}}
NET=nemo
DEMOVM=cloonix-image-minimal-qemux86.wic.qcow2
CONFIGS=../cloonix_emqx/basic

cloonix_cli ${NET} kil
cloonix_net ${NET}
cloonix_gui ${NET}
cloonix_cli ${NET} add kvm bird0 ram=800 cpu=2 dpdk=0 sock=3 hwsim=0 ${DEMOVM} &
cloonix_cli ${NET} add kvm bird1 ram=800 cpu=2 dpdk=0 sock=3 hwsim=0 ${DEMOVM} &

cloonix_ssh ${NET} bird0 "echo" 2>/dev/null
cloonix_ssh ${NET} bird1 "echo" 2>/dev/null

cloonix_cli ${NET} add lan bird0 0 lan1
cloonix_cli ${NET} add lan bird0 1 lan2
cloonix_cli ${NET} add lan bird0 2 ser1

cloonix_cli ${NET} add lan bird1 0 lan1
cloonix_cli ${NET} add lan bird1 1 lan2
cloonix_cli ${NET} add lan bird1 2 ser1

cloonix_scp ${NET} ${CONFIGS}/bird1.config bird1:/usr/lib/emqx/etc/emqx.config
cloonix_scp ${NET} ${CONFIGS}/bird2.config bird2:/usr/lib/emqx/etc/emqx.config

cloonix_ssh ${NET} bird1 "systemctl enable emqx.service"
cloonix_ssh ${NET} bird1 "systemctl enable emqx.service"

cloonix_ssh ${NET} bird1 "systemctl restart emqx"
cloonix_ssh ${NET} bird1 "systemctl restart emqx"
{{< / highlight >}}

That script is very interesting. All that we need to do is describe the steps needed to create the environment. In short we have:

* line 5, do a cleanup
* line 6 and 7 starts a network and a graphic interface
* line 8-9 creates two KVM machines with 3 network interfaces
* line 11-12 tries to connect over ssh to check if the machines are up
* line 14-20 creates and connects the network
* line 22-29 tries to configure and start an application

The commands `cloonix_ssh` and `cloonix_scp` are very handy to fix and configure the machine without need a working network.

Check the working script here: [joaohf/cloonix_emqx](https://github.com/joaohf/cloonix_emqx/tree/master/basic).

After all that commands, the following components have been appeared in the cloonix_gui:

{{< figure src="/media/cloonix_basic_gui.jpg" alt="Cloonix with a basic network setup" >}}

As a plus, cloonix has a way to attach a sniffer and listen packets.

{{< figure src="/media/cloonix_basic_with_snf.jpg" alt="Cloonix with a basic network setup and wireshark" >}}

## What else I can do with that ?

The [meta-cloonix](https://github.com/joaohf/meta-cloonix) Yocto layer allows use any image, created with Yocto, to work with cloonix network emulator. You can install any software, create a cloonix script and test how is the software behaviour inside a controlled network.

Take a look over the [cloonix repository](https://github.com/clownix/cloonix) and the official [website](https://clownix.net/) to get more ideas about what else you can do.
