---
layout: post
title: "Shef Tips and Tricks: Stepping Through Chef-client Runs with Shef"
date: 2012-01-28 15:13
comments: true
categories: shef chef
---

Recently, I've used [Shef](http://wiki.opscode.com/display/chef/Shef),
the interactive Chef console, to get to the bottom of some rather
tricky problems.  In the process, I developed a number of tricks and
tools for productively debugging problems using Shef.  This is the first
in a series of articles I'd like to write on the most useful of these
tips.

In this article I will discuss how I use Shef and a handful of
additional functions to debug Chef-client runs by stepping through a
node's run list, breaking before or after specific resources in order
to inspect the state of the system.  Note that this is not intended
to be an introduction to Shef.  For that, you should head over to the
[Chef Wiki](http://wiki.opscode.com/display/chef/Getting+Started+with+Shef).

# Setup

Some of the functions I mention in this article are in the `shef/`
directory of my [knife-hacks](https://github.com/stevendanna/knife-hacks) repository.  If you'd like to use them,
you can download this repository from Github:

```bash
mkdir ~/src
cd ~/src
git clone git://github.com/stevendanna/knife-hacks.git
```
To make it easy to include files from this repository inside Shef, you
can use the following in the relevant configuration file:

```ruby
unless defined?(Shef).nil?
  $: << File.expand_path("~/src/knife-hacks/")
end
```

All of the examples I will be doing will be on my local workstation.
For ease, I have placed this in my `knife.rb` file, which I will
pass to Shef using the `-c` command line option.  However, a similar
approach will work within a `client.rb` for a node or a custom `shef.rb`
file.

# Introduction
Shef is an interactive console for Chef.  Essentially, it is IRB with
support for recipe and attribute syntax and a number of Chef-relevant
functions.

Here's a typical Shef session that you might find in an introduction
to Shef.  We add some resources, including a *breakpoint* resource and
start a chef-run.

```
chef > recipe
chef:recipe > echo off
chef:recipe > log "a"
chef:recipe > breakpoint "STOP"
chef:recipe > log "b"
chef:recipe > run_chef
DEBUG: Processing log[a] on sdanna
INFO:  Processing log[a] action write ((irb#1) line 3)
INFO:  a
DEBUG: Processing [/home/sdanna/.rvm/gems/ruby-1.9.3-p0/gems/chef-0.10.8/lib/chef/mixin/recipe_definition_dsl_core.rb:61:in `new'] on sdanna
INFO:  Processing [/home/sdanna/.rvm/gems/ruby-1.9.3-p0/gems/chef-0.10.8/lib/chef/mixin/recipe_definition_dsl_core.rb:61:in `new'] action break ((irb#1) line
```

The breakpoint stops the chef-client run, allowing you to
investigate the state of the system and then resume the run with `chef_run.resume`

```
chef:recipe > chef_run.resume
INFO:  Processing log[b] action write ((irb#1) line 6)
INFO:  b
```

With these basic features, Shef can be great for running quick tests
of recipe code.  However, it is not immediately clear how you get from
here to being able to step through an actual chef-client run.  To
effectively step through a chef-client run, we need to be able to:

* Load resources from the recipes in the node's run_list, and
* Insert breakpoints between the loaded resources.

The remainder of this article covers how to accomplish these two
tasks.

## Loading Resources from the RunList

Invoking Shef with the `-z` argument enables client-mode, forcing it to
download the relevant recipes from the node's run_list just as
chef-client would, but it does not processes these recipes and add
their resources to the resource collection.

Because Shef's recipe context allows you to use the recipe DSL, we can
add the resources from an individual recipe using the `include_recipe`
function.  We can only use `include_recipe` on recipes that Shef
downloaded when we started it up.

```
chef:recipe > resources
[]
chef:recipe > include_recipe "ssh_known_hosts"
chef:recipe > resources
["template[/etc/ssh/ssh_known_hosts]"]
```

In order to load all of the recipes we need to iterate over a list of
all the recipes in the run_list and call `include_recipe` on each of
them.  Since the run_list can contain roles, we also need to ensure we
get the expanded run_list.  There are a number of ways to get the
expanded run_list.  Here is one:

```ruby
node.run_list.expand(node.chef_environment).recipes.each do |r|
  include_recipe r
end
```

This simple bit of Ruby is much like the *resource compilation* phase of
a chef-client run.  Just as within a chef-client run, you may
encounter an error as you try to include the recipes. In a future
article, I may discuss better ways to step through this compilation
phase.

Since I'd rather not type the above every time I want to debug a
chef-client run, I encapsulated this into a function within my
ShefExtras library: [`load_node_run_list`](https://github.com/stevendanna/knife-hacks/blob/master/shef/extras.rb#L11).

## Inserting Breakpoints between Resources

A large run_list can easily contain hundreds of resources.  While we
could step through the resources one-by-one, we often want to run
through a large number of the resources, and then stop just before a
resource that exhibiting some errant behavior.To do this, we need a
way to insert breakpoints between the resources we are loading from
recipes.

Since "breakpoint" is a fully-fledged Chef resource, we could place
the breakpoint resources directly in the relevant recipe. Within a
normal chef-client run, the breakpoint resources will have no effect,
allowing us to do this without much fear of endangering other
nodes using the same recipe.

However, I've found that it is more useful to be able to add the
breakpoints via Shef, since we will often want to add new breakpoints
as we gain new information.

To accomplish this, I've created an `insert_break` function to do just
that within my ShefExtras library.  Since it is a bit uglier than I
would like and depends on mucking about in the depths of Chef's data
structures, I am not going to walk through how the function works.
However, here is an example of how to use it:

* First, we load the ShefExtras library and switch to recipe mode.
```
chef > echo off
chef > require 'shef/extras'
chef > ShefExtras.load
INFO: ShefExtras loaded!
chef > recipe
```

* Next, we load the resources from the recipe in the node's run list
and insert a break point before one of them.

```
chef:recipe > load_node_run_list
chef:recipe > resources
["log[Please set the set_fqdn attribute to desired hostname]",
"template[/etc/ssh/ssh_known_hosts]"]
chef:recipe > insert_break :before, "template[/etc/ssh/ssh_known_hosts]"
INFO:  Breakpoint added before template[/etc/ssh/ssh_known_hosts]
chef:recipe > ordered_resources
=> ["log[Please set the set_fqdn attribute to desired hostname]",
"break[break-before-template[/etc/ssh/ssh_known_hosts]]",
"template[/etc/ssh/ssh_known_hosts]"]
```

* Finally, we run chef.

```
chef:recipe > run_chef
DEBUG: Processing log[Please set the set_fqdn attribute to desired hostname] on sdanna
INFO:  Processing log[Please set the set_fqdn attribute to desired hostname] action write (hostname::default line 64)
WARN:  Please set the set_fqdn attribute to desired hostname
DEBUG: Processing break[break-before-template[/etc/ssh/ssh_known_hosts]] on sdanna
INFO:  Processing break[break-before-template[/etc/ssh/ssh_known_hosts]] action break (/home/sdanna/src/knife-hacks/shef/extras.rb line 38)
```

As you can see, the chef-client run halts before running the template
resources.  At this point we could investigate elements of the system
state that we believe may be having an effect on the template
resource. When we are ready to continue, we can call `chef_run.resume`
to continue the run.

In this example, I've used three custom functions from the ShefExtras
module:

 * `load_node_run_list` loads the resources from the
node's run list in the same way as we did in the previous section.

 * `ordered_resources` returns a list of the resources in the order that
chef-client would run them.

 * `insert_break(preposition, resource)` will place a break resource
either before or after the given resource.

# When Do I Use This

Using these tools to step through the resource list is particularly
useful in the following types of situations:

* The run list contains recipes which are heavily dependent on execute
  or script resources that may not be behaving as you expect.
* A recipe makes significant run time modifications of node attributes.
* A recipe's behavior non-trivially branches based on pieces of the
  system's state that is not easily observed before or after the
  chef-client run has completed.

While I still prefer to start with reading the relevant recipe and
reasoning about what will happen, having the ability to quickly run a
portion of the recipe and confirm a hypothesis about the state of the
system has proven incredibly valuable.

# More to Come
Future articles in this series may cover:

* Customizing your Shef configuration,
* Using Shef to make bulk changes to node data,
* Analyzing API responses using Shef,
* Running Shef as an inferior process in Emacs, and
* Debugging errors with resource compilation.
