---
title: CircleCI
date: 2019-11-24T14:00:00-05:00
toc: true
featured_image: 'media/yocto-way3.jpg'
---

# Introduction

[circleci](https://circleci.com/) is a kind of Continuous Integration platform that you can build your own build pipeline. The platform provides nice ways to integrate with existing code repositories as well good features to use. It is free to use when doing some Open Source project and additional costs when use for commercial or private purposes.

Before test circleci, I've tried another platform called [travisci](https://travis-ci.com/). But I was looking to a feature which I could use to cache some objects easily and restore them later.

## A basic working example

First of all, create a directory called `.circleci`, where all files and configurations
related to circleci will be hold. After that, create a new circleci config file called `.circleci/config.yml` and copy the below configuration to the new file.

{{< highlight yaml "linenos=inline,linenostart=1" >}}
version: 2.1

jobs:
  build:
    docker:
      - image: crops/yocto:ubuntu-19.04-builder
        environment:
          LC_ALL: en_US.UTF-8
          LANG: en_US.UTF-8
          LANGUAGE: en_US.UTF-8

    steps:
      - checkout
      - restore_cache:
          keys:
            - sstate-test-{{ .Branch }}-{{ .Revision }}
            - sstate-test-{{ .Branch }}-
            - sstate-test-master-
            - sstate-test-
      - run:
          name: Clone poky
          command: git clone -b warrior --single-branch git://git.yoctoproject.org/poky
      - run:
          name: Clone meta-oe
          command: git clone -b warrior --single-branch https://github.com/openembedded/meta-openembedded.git meta-oe
      - run:
          name: Link meta-erlang
          command: ln -sf . meta-erlang
      - run:
          name: Initialize build directory
          command: TEMPLATECONF=../.circleci source poky/oe-init-build-env build
      - run:
          name: Build erlang
          command: source poky/oe-init-build-env build && bitbake erlang
          no_output_timeout: 50m
      - run:
          name: Build emqx
          command: source poky/oe-init-build-env build && bitbake emqx
          no_output_timeout: 50m

      - save_cache:
          key: sstate-test-{{ .Branch }}-{{ .Revision }}
          paths:
            - "build/sstate-cache/"

workflows:
  workflow:
    jobs:
      - build
{{< / highlight >}}


There are a few notes about the configuration:

* A docker image called [crops/yocto](https://hub.docker.com/r/crops/poky) will be used. This is very important because we have garanties that the correct and compatible linux distro will be use during the build

* _steps_ are used to declare all the procedures necessary to build. Basic you need to descript your commands when use yocto/bitbake

* The _restore\_cache_ and _save\_cache_ are two circleci features which allow use of a cache layer yo speed up the build

### Providing Yocto CI configuration

Most of the time, during the CI, we need to have additional configurations that Yocto will use in order to build the desired recipes and images. An easy way to do this is configure the `local.conf` and `bblayers.conf` using the [TEMPLATECONF](https://www.yoctoproject.org/docs/2.7/ref-manual/ref-manual.html#structure-build-conf-local.conf).

With _TEMPLATECONF_ we can pass a directory which have config sample files and bitbake will use these files as the template to `local.conf` and `bblayers.conf`. That way we can configure our CI build values.

Examples:

* `.circleci/local.conf.sample`: any custom configuration should be place here
{{< highlight bash >}}
MACHINE ??= "qemuarm"
DISTRO ?= "poky"
PACKAGE_CLASSES ?= "package_ipk"
EXTRA_IMAGE_FEATURES ?= "debug-tweaks"
USER_CLASSES ?= "buildstats image-mklibs image-prelink"
PATCHRESOLVE = "noop"
BB_DISKMON_DIRS ??= "\
    STOPTASKS,${TMPDIR},1G,100K \
    STOPTASKS,${DL_DIR},1G,100K \
    STOPTASKS,${SSTATE_DIR},1G,100K \
    STOPTASKS,/tmp,100M,100K \
    ABORT,${TMPDIR},100M,1K \
    ABORT,${DL_DIR},100M,1K \
    ABORT,${SSTATE_DIR},100M,1K \
    ABORT,/tmp,10M,1K"
PACKAGECONFIG_append_pn-qemu-system-native = " sdl"
PACKAGECONFIG_append_pn-nativesdk-qemu = " sdl"
CONF_VERSION = "1"

SSTATE_DIR ?= "${TOPDIR}/sstate-cache"
SSTATE_MIRRORS = "\
file://.* http://sstate.yoctoproject.org/dev/PATH;downloadfilename=PATH \n \
file://.* http://sstate.yoctoproject.org/2.6/PATH;downloadfilename=PATH \n \
file://.* http://sstate.yoctoproject.org/2.7.2/PATH;downloadfilename=PATH \n \
"
{{< / highlight >}}

* `.circleci/bblayers.conf.sample`: where all the layers are declared:
{{< highlight bash >}}
POKY_BBLAYERS_CONF_VERSION = "2"
BBPATH = "${TOPDIR}"
BBFILES ?= ""
BBLAYERS ?= " \
  ##OEROOT##/meta \
  ##OEROOT##/meta-poky \
  ##OEROOT##/meta-yocto-bsp \
  ##OEROOT##/../meta-erlang \
  ##OEROOT##/../meta-oe/meta-oe \
  "
{{< / highlight >}}

### The final directory layout

{{< highlight bash >}}
.circleci/
.circleci/local.conf.sample
.circleci/bblayers.conf.sample
.circleci/config.yml
{{< / highlight >}}
