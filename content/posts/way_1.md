---
title: How make sure that your layer is compatible and up-to-date, meta-erlang case
date: 2010-10-05T12:00:00-05:00
toc: true
---

# A tool to check layer

I was reading the Yocto Project development manual when I found a topic ([Making Sure Your Layer is Compatible With Yocto Project](https://www.yoctoproject.org/docs/latest/dev-manual/dev-manual.html#making-sure-your-layer-is-compatible-with-yocto-project)) about how to use a tool called _yocto-check-layer_ to perform basics checks and detect if a specific layer is compatible or needs adjustments.

So I tried to run that tool against the _meta-erlang_ layer:

{{< highlight bash >}}
joaohf@porco:~/tmp/poky$ source oe-init-build-env ~/tmp/yp/build
joaohf@porco:~/tmp/yp$ yocto-check-layer ~/tmp/meta-erlang
INFO: Detected layers:
INFO: meta-erlang: LayerType.SOFTWARE, /home/joaohf/tmp/meta-erlang
INFO: 
INFO: Setting up for meta-erlang(LayerType.SOFTWARE), /home/joaohf/tmp/meta-erlang
INFO: Getting initial bitbake variables ...
INFO: Getting initial signatures ...
INFO: Adding layer meta-erlang
INFO: Starting to analyze: meta-erlang
INFO: ----------------------------------------------------------------------
INFO: skipped "BSPCheckLayer: Layer meta-erlang isn't BSP one."
INFO: test_layerseries_compat (common.CommonCheckLayer)
INFO:  ... ok
INFO: test_parse (common.CommonCheckLayer)
INFO:  ... ok
INFO: test_readme (common.CommonCheckLayer)
INFO:  ... ok
INFO: test_show_environment (common.CommonCheckLayer)
INFO:  ... ok
INFO: test_signatures (common.CommonCheckLayer)
INFO:  ... ok
INFO: test_world (common.CommonCheckLayer)
INFO:  ... ok
INFO: skipped "DistroCheckLayer: Layer meta-erlang isn't Distro one."
INFO: ----------------------------------------------------------------------
INFO: Ran 6 tests in 128.584s
INFO: OK
INFO:  (skipped=2)
INFO: 
INFO: Summary of results:
INFO: 
INFO: meta-erlang ... PASS
{{< / highlight >}}

If you have a layer and/or is a Yocto Project member, you should consider to use that tool to check your layer.

As a Yocto Project member, a layer can be considered official and compatible with Yocto Project after sending the results to  [Compatible Registration program](https://www.yoctoproject.org/webform/yocto-project-compatible-registration) and get an approval.
