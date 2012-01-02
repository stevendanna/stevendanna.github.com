---
layout: post
title: "Snow and Chef"
date: 2011-12-26 09:32
comments: true
categories: R chef mpi
---
One of my major interests is programming in R.  I took a few days off
for Christmas break, and in between talking with family, watching old
movies, opening presents, and spreading Christmas cheer, I decided it
would be fun to use Chef to create a parallel computing environment
suitable for use with R.  As part of this project, I wrote:

* A Chef cookbook to install and configure R.
* An R package provider to easily install packages from CRAN.
* A Chef cookbook to install and configure MPI.

This blog post outlines some of the highlights of this fun little
project. It isn't a step-by-step tutorial for creating such
an environment.  I assume that already have:

* General familiarity with R and Chef.  I talk about cookbooks,
  recipes, and roles and assume you know what to do with them.
* A means of provisioning computation nodes, each already configured
  with hostnames that are resolvable to network addresses by other
  computation nodes.

I used vagrant to provision a few VMs for testing these recipes and a
small set of vagrant-specific cookbooks to ensure that each node had a
resolvable hostname.

# R Cookbook
The first item every node in the cluster will need is R.  The R
cookbook on the Opscode Community site is a bit out of date.  I've
created a newer version that can be found here:

[http://github.com/stevendanna/cookbook-r](http://github.com/stevendanna/cookbook-r)

This cookbook does the following:

* Installs R from either the CRAN APT repository or from source depending
  on the platform.
* Defines a system-wide default CRAN mirror using an Rprofile.site
  template.
* Contains an R package provider that can install R packages available
  on CRAN.

Currently this cookbook is linux-centric and best suited for use on
Ubuntu or Debian.

## R Package Provider
My R parallel computing environment relies on the 'snow' and 'Rmpi'
packages to manage communication with computation nodes.  Further, I
often need additional packages from CRAN when working with my cluster.
The R package provider in my R cookbook allows for easy, automated
installation of R packages.  It was written using Chef's Light-weight
Resource and Provider DSL.  The current version provides the minimal
necessary functionality:

``` ruby "R package provider.  Corresponding resource description not shown."
action :install do
  execute "Install R Package" do
    command r_package_install(new_resource.package)
    not_if r_package_is_installed(new_resource.package)
  end
end

action :upgrade do
  execute "Upgrade R Package" do
    command r_package_install(new_resource.package)
  end
end

action :remove do
  r_package_remove = "remove.packages('#{new_resource.package}')"
  execute "Remove R Package" do
    command "echo \"#{r_package_remove}\" | R --no-save --no-restore -q"
    only_if r_package_is_installed(new_resource.package)
  end
end


# The following helper functions construct strings that can be run as
# Bash commands. For example, as the input of not_if or only_if
# statements

def r_package_is_installed(package_name)
  r_code = "if (any(installed.packages()[,1] == '#{package_name}')) { quit('no', 0, FALSE) }; quit('no', 1, FALSE)"
  "echo \"#{r_code}\" | R --no-save --no-restore -q"
end

def r_package_install(package_name)
  r_code = "install.packages('#{package_name}')"
  "echo \"#{r_code}\" | R --no-save --no-restore -q"
end
```

It supports three actions: `:install`, `:upgrade`, and
`:remove`; however, it is not very feature-rich.  A few features I
would like to add include:

* The ability to specify the CRAN mirror or repository.
* The ability to install local packages
* The ability to install packages on a per-user basis.

Despite these shortcomings, the current provider works well enough for
my purposes and is a good example of Chef's high-level of
'Whip-It-Up-itude.'  Within a few minutes, I was able to go from some
quick-and-dirty R code to a Chef provider that I can iteratively
improve as my needs become more complex.  While I did not include it
in the repository linked above, I was able to create a small `R::snow`
recipe that does little more than use this provider:

``` ruby
R_package "snow"

case node['R']['snow']['cluster_type']
when 'mpi'
  include_recipe 'mpi'
  R_package 'Rmpi'
end
```

# MPI Cookbook
The R package 'snow' can use various backends to communicate with
computation nodes.  The default method for most commands is "Message
Passing Interface" (MPI).  MPI is also generally useful outside of R
as you can easily use MPI utilities such as mpirun to execute a
command on multiple computation nodes and use MPI's C libraries to
write programs designed for parallel computation.

MPI has multiple implementation.  I've written an MPI cookbook that
can install [openmpi](http://www.open-mpi.org/) on Ubuntu.  It can be found here:

[https://github.com/stevendanna/cookbook-mpi](https://github.com/stevendanna/cookbook-mpi)

This cookbook does the following:

* Installs openmpi on Ubuntu
* Constructs a list of computation nodes within your environment using
  search and then renders a default hostfile that is used by MPI
  commands such as mpirun.
* Sets basic configuration options that I have found useful.

While MPI does not natively contain the concepts of master and slaves
(every MPI node can talk to any other MPI node), I found it useful to
include the concept of a node's "MPI role."  I use this role to
populate the default hostfile, since there are often cases where you
might want MPI installed on a machine within your environment but
would not want that machine being used for computation.

# Putting it all Together
I put this all together using two roles: snow\_master and snow\_slave

``` ruby roles/snow_master.json
{
    "name" : "snow_master",
    "json_class" : "Chef::Role",
    "chef_type" : "role",
    "run_list" : [ "recipe[R]", "recipe[R::snow]", "recipe[mpi]", "recipe[ssh_known_hosts]" ],
    "default_attributes" : {
        "mpi" : {
            "role": "master"
        }
    }
}
```

``` ruby roles/snow_slave.json
{
    "name" : "snow_slave",
    "json_class" : "Chef::Role",
    "chef_type" : "role",
    "run_list" : [ "recipe[R]", "recipe[R::snow]", "recipe[mpi]", "recipe[ssh_known_hosts]" ]
}
```

By applying the snow\_slave role to all of the nodes I wanted within
my computation cluster and the snow\_master role to the machine I will
use to launch jobs in my cluster, I can quickly bring up a new cluster
of machines suitable for doing parallel computation in R using snow.

The ssh\_known\_hosts recipe is directly from the Opscode community
site and helps make ssh connections between the nodes a bit smoother.

# Twelve Days of Chistmas

As an example of how easy it is to do simple parallel computation
using snow, consider the following Chistmas-themed R function.

``` r
twelve_days_of_christmas <- function (n) {
  if (!any(1:12 == n)) stop("Not a day of Chirstmas!")
  LYRICS <- c("a Partridge in a Pear Tree.",
              "Two Turtle Doves",
              "Three French Hens",
              "Four Colly Birds",
              "Five Golden Rings",
              "Six Geese-a-Laying",
              "Seven Swans-a-Swimming",
              "Eight Maids-a-Milking",
              "Nine Ladies Dancing",
              "Ten Lords-a-Leaping",
              "Eleven Pipers Piping",
              "Twleve Drummers Drumming")
  append = c("st", "nd", "rd", rep("th",9))
  if (n > 1) LYRICS[1] = paste("and", LYRICS[1])
  paste("On the",
        paste(n, append[n], sep=""),
        "day of Christmas, my true love gave to me",
        paste(LYRICS[n:1], collapse=", "))
}
```

Assuming chef-client has been run on all of our nodes, we can simply
log into our 'snow master', boot up R, and run the following code:

``` r
> library('snow')
> source('twelve_days.r')
> cl <- makeCluster(2)
> clusterApply(cl, 1:12, twelve_days_of_christmas)
[[1]]
[1] "On the 1st day of Christmas, my true love gave to me a Partridge in a Pear Tree."

[[2]]
[1] "On the 2nd day of Christmas, my true love gave to me Two Turtle Doves, and a Partridge in a Pear Tree."

[[3]]
[1] "On the 3rd day of Christmas, my true love gave to me Three French Hens, Two Turtle Doves, and a Partridge in a Pear Tree."

[[4]]
[1] "On the 4th day of Christmas, my true love gave to me Four Colly Birds, Three French Hens, Two Turtle Doves, and a Partridge in a Pear Tree."

[[5]]
[1] "On the 5th day of Christmas, my true love gave to me Five Goldedn Rings, Four Colly Birds, Three French Hens, Two Turtle Doves, and a Partridge in a Pear Tree."

[[6]]
[1] "On the 6th day of Christmas, my true love gave to me Six Geese-a-Laying, Five Goldedn Rings, Four Colly Birds, Three French Hens, Two Turtle Doves, and a Partridge in a Pear Tree."

[[7]]
[1] "On the 7th day of Christmas, my true love gave to me Seven Swans-a-Swimming, Six Geese-a-Laying, Five Goldedn Rings, Four Colly Birds, Three French Hens, Two Turtle Doves, and a Partridge in a Pear Tree."

[[8]]
[1] "On the 8th day of Christmas, my true love gave to me Eight Maids-a-Milking, Seven Swans-a-Swimming, Six Geese-a-Laying, Five Goldedn Rings, Four Colly Birds, Three French Hens, Two Turtle Doves, and a Partridge in a Pear Tree."

[[9]]
[1] "On the 9th day of Christmas, my true love gave to me Nine Ladies Dancing, Eight Maids-a-Milking, Seven Swans-a-Swimming, Six Geese-a-Laying, Five Goldedn Rings, Four Colly Birds, Three French Hens, Two Turtle Doves, and a Partridge in a Pear Tree."

[[10]]
[1] "On the 10th day of Christmas, my true love gave to me Ten Lords-a-Leaping, Nine Ladies Dancing, Eight Maids-a-Milking, Seven Swans-a-Swimming, Six Geese-a-Laying, Five Goldedn Rings, Four Colly Birds, Three French Hens, Two Turtle Doves, and a Partridge in a Pear Tree."

[[11]]
[1] "On the 11th day of Christmas, my true love gave to me Eleven Pipers Piping, Ten Lords-a-Leaping, Nine Ladies Dancing, Eight Maids-a-Milking, Seven Swans-a-Swimming, Six Geese-a-Laying, Five Goldedn Rings, Four Colly Birds, Three French Hens, Two Turtle Doves, and a Partridge in a Pear Tree."

[[12]]
[1] "On the 12th day of Christmas, my true love gave to me Twleve Drummers Drumming, Eleven Pipers Piping, Ten Lords-a-Leaping, Nine Ladies Dancing, Eight Maids-a-Milking, Seven Swans-a-Swimming, Six Geese-a-Laying, Five Goldedn Rings, Four Colly Birds, Three French Hens, Two Turtle Doves, and a Partridge in a Pear Tree."
```

# Conclusion

Overall this was a fun project to do over break.  It took a handful of
hours to complete, most of which was dedicated to a few problems not
discussed in this post:

* Making hostnames resolvable and other network oddities within my
  Vagrant test environment.
* Deciding which implementation of MPI to use.

I personally would like to see further work done around using Chef to
manage large R environments. Some work I hope to do in the not to
distant future includes:

* Expand the R cookbook to better support OS X.
* Add support to detect the latest version when doing a source install.
* Expand the R package provider to include more of the features available
  when installing package from within R.
* Write an R Ohai plugin that would collect information about:
    * R version and capabilities
    * R packages installed

Happy Holidays!
