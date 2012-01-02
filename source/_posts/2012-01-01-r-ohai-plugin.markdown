---
layout: post
title: "R ohai plugin"
date: 2012-01-01 15:44
comments: true
categories: R Chef
---
[Ohai](https://github.com/opscode/ohai) is a library used by Chef to collect information about nodes
within your infrastructure.  Information is collected by a set of Ohai
plugins, most of which parse the output of system commands.

During lunch on Friday, I create a simple Ohai plugin to collect some basic
information about R. You can find the plugin here:

  [https://github.com/stevendanna/ohai-plugins/blob/master/r.rb](https://github.com/stevendanna/ohai-plugins/blob/master/r.rb)

Currently, this plugin collects:

* The version of R,
* The installed packages, and
* The output of `capabilities()`

To use this plugin right away, the easiest course of action is to use
the [Opscode Ohai cookbook](http://community.opscode.com/cookbooks/ohai).  From your chef repository:

``` bash
knife cookbook site install ohai
wget https://raw.github.com/stevendanna/ohai-plugins/master/r.rb
mv r.rb cookbooks/ohai/files/default/plugins/
knife cookbook upload ohai
# Add the recipe to the run list of the relevant nodes
# or roles
knife node run_list add NODENAME ohai
# The ohai plugin will run on the next chef-client run
knife ssh 'name:NODENAME'
```

In order to avoid unnecessary runs of Ohai within a chef-client run
you can add the following line to the `client.rb` configuration file on your nodes:

``` ruby
Ohai::Config[:plugin_path] << '/etc/chef/ohai_plugins'
```

If you are using Opscode's chef-client cookbook, this will already be
taken care of for you.

The following is an example of the information it collects:

```
> knife node show snow-master -a languages.r -Fj
{
  "languages.r": {
    "capabilities": {
      "tiff": true,
      "cledit": false,
      "tcltk": true,
      "X11": false,
      "sockets": true,
      "fifo": true,
      "iconv": true,
      "cairo": true,
      "png": true,
      "http/ftp": true,
      "libxml": true,
      "jpeg": true,
      "aqua": false,
      "NLS": true,
      "profmem": true
    },
    "packages": [
      {
        "name": "Rmpi",
        "version": "0.5-9",
        "built": "2.14.1"
      },
      {
        "name": "snow",
        "version": "0.3-8",
        "built": "2.14.1"
      },
      {
        "name": "KernSmooth",
        "version": "2.23-7",
        "built": "2.14.0"
      },
[ ... SNIP MANY MORE PACKAGE ...]
      {
        "name": "boot",
        "version": "1.3-3",
        "built": "2.14.0"
      },
      {
        "name": "utils",
        "version": "2.14.1",
        "built": "2.14.1"
      }
    ],
    "version": "2.14.1"
  }
}

```

Also, Happy New Year!
