---
title: meta-cloonix
date: 2010-10-05T12:00:00-05:00
toc: true
---

## Introduction

What is cloonix ?

## Creating cloonix images

How a normal image is used ?

## meta-cloonix

What is meta-cloonix ?

## Using meta-clotonix layer to create images

Clone `meta-cloonix` from https://www.github.com/joaohf/meta-cloonix repository and enable it in the file `conf/bblayers.conf`. See how to do this in [Enabling Your Layer](https://www.yoctoproject.org/docs/latest/dev-manual/dev-manual.html#enabling-your-layer).

In the file `conf/local.conf` we need to add two additional configurations:

Enable the cloonix DISTRO_FEATURE:

DISTRO_FEATURES_append = " cloonix"

Use the cloonix specific WKS_FILE:

WKS_FILE = "cloonix-qemux86-directdisk.wks"

These two configurations will enable an image which been created with:

* correct linux kernel append flags
* special configuration to enable inittab spawing a hvc0 console

meta-cloonix provides an image, based com _core-image_minimal_, which we can use to build a basic image to test the Cloonix environment:

bitbake cloonix-image-minimal

After the build, copy the image file to CLOONIX_BULK directory:

tmp/deploy/images/qemux86/cloonix-image-minimal-qemux86.wic.qcow2 $HOME/cloonix_data/bulk

Right now we created a image suitable to use with cloonix with Yocto. Remember that you are free to install any kind of extra software and also enable other layers.

## Building a network using cloonix

Now, we will work only with Cloonix command and graphic interface to build a small network within some nodes sending packets and a protocol sniffer, allowing seen what is going on in the network.

In this post we will build a script which does all the needed configuration.

That is the network:

          +---+
          |snf|
          +-+-+
            |
+---+       |       +---+
|kvm+-------+-------+kvm|
+---+      lan      +---+

See more about the cloonix objects used in this example here https://clownix.net/doc_stored/build-03-04/html/doc/objects.html

So far, our network has:

* two kvm guests (using the cloonix-image-minimal)
* a lan to connect these two kvm
* a sniffer listening all the traffic between the hosts

Above, the following cloonix script:

## What else I can do with this ?

The `meta-cloonix` Yocto layer is just allow you use any image, created with Yocto, to work with cloonix network emulator. You can install any software, create a cloonix script and test how is the software behaviour inside a network.

Take a look over the [cloonix repository](https://github.com/clownix/cloonix) and [website](https://clownix.net/) to get more ideas what else you can do.
