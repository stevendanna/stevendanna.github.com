---
layout: post
title: "gsd: A small utility to get stuff done"
date: 2011-11-06 09:48
comments: true
categories:
---

In preparation for moving more of my personal laptop's setup into git
and Chef, I spent a few minutes this morning taking stock of what I
had floating around my home folder on my personal laptop.

I found a small script I wrote when working on my thesis to help me
avoid distractions.  Since I thought others might be interested, I put
it up on Github and added a bit more user friendliness:

*[gsd: A small utility to get stuff done](https://github.com/stevendanna/gsd)*

`gsd` helps you get work done by replacing your host file with a
custom host file kept in `~/.gsd/`.  The custom host file prevents you
from visiting sites that distract you from the work you need to get done.

To start using the filter, simply clone the git repo and install the
script

    > git clone git@github.com:stevendanna/gsd.git
    > cd gsd
    > sudo make install

Then, enable the filter

    > sudo gsd enable

Finally, add sites to the filter and reload when you find yourself
being distracted by them:

    > gsd add lkml.org
    > sudo gsd reload

With a bit of searching, one can find a number of these types of
scripts online.  My favorite that I've come across so far is [Get Shit Done](https://github.com/leftnode/get-shit-done), which works on
Windows and includes some logic to restart your systems networking
services.

