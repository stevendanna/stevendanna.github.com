---
layout: post
title: "Cookbook Release: xinetd 0.9.0"
date: 2013-04-20 10:44
comments: true
categories: chef cookbooks xinetd
---

Earlier this week I released version 0.9.0 of my
[xinetd cookbook](https://github.com/stevendanna/cookbook-xinetd).
This cookbook provides:

- An LWRP for managing xinetd services,
- Attribute driven configuration of xinetd.conf, and
- a recipe for enabling the internal xinetd-provided services
  (chargen, daytime, discard, echo, time, and tcpmux-server).

This cookbook is not yet on the Opscode Community site as I am working
with the maintainer of the currently uploaded xinetd cookbook to
ensure we don't break anyone using the existing cookbook.

For more details about this cookbook, please see the [README](https://github.com/stevendanna/cookbook-xinetd/blob/master/README.md).  If you
are interested in contributing to this cookbook, pull requests are
accepted on Github.
