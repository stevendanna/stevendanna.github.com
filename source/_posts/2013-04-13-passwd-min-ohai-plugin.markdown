---
layout: post
title: "Passwd_min Ohai Plugin"
date: 2013-04-13 13:23
comments: true
categories: chef ohai
---

[passwd_min](https://github.com/stevendanna/ohai-plugins/blob/master/plugins/passwd_min.rb)
is an Ohai plugin that parses /etc/passwd and /etc/group and returns
user and group information in a format that matches the passwd plugin
shipped in Ohai.

This plugin may be useful to Chef users who have alternate
password databases (such as LDAP or NIS) configured and have had to
disable the default passwd plugin to avoid storing their organization's
entire LDAP directory in their node objects.

## Background

The passwd plugin shipped in Ohai uses Ruby's Etc module to popular
the `node['etc']['passwd']` and `node['etc']['group']` attributes.

If one follows the code for the Etc module far enough, she will find
that this information is obtained by calling the getpwent() system
call:

```c ext/etc/etc.cnn
passwd_iterate(void)
{
    struct passwd *pw;

    setpwent();
    while (pw = getpwent()) {
        rb_yield(setup_passwd(pw));
    }
    return Qnil;
}
```

According to the Linux Manual Page for getwpent():

> The getpwent() function returns a pointer to a structure containing
  the broken-out fields of a record from the password database (e.g.,
  the local password file /etc/passwd, NIS, and LDAP). The first time
  getpwent() is called, it returns the first entry; thereafter, it
  returns successive entries.

Thus, for users who have have LDAP or NIS configured,
`node['etc']['passwd']` will contain more than simply the contents of
`/etc/passwd`.  For some users, the data the default passwd plugin
collects will produce large node objects.  These large node objects
place additional load on the Chef Server and other chef clients that
have to process those objects later.

## Using the passwd_min plugin

The easiest way to distribute the passwd_min plugin is via the
[Opscode Ohai cookbook](http://community.opscode.com/cookbooks/ohai).  From your Chef Repository:

```bash
knife cookbook site install ohai
wget https://raw.github.com/stevendanna/ohai-plugins/master/passwd_min.rb
mv passwd_min.rb cookbooks/ohai/files/default/plugins/
knife cookbook upload ohai
# Add the recipe to the run list of the relevant nodes
# or roles
knife node run_list add NODENAME ohai
```

You will also want to disable the default passwd plugin by placing the
following in /etc/chef/client.rb on the node:

     Ohai::Config[:disabled_plugins] = ["passwd"]

Note that if you use the Chef-client Cookbook, this can be done via an attribute.

Bug reports and improvements happily accepted on Github.
