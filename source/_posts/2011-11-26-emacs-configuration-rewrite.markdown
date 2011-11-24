---
layout: post
title: "Emacs Configuration Rewrite"
date: 2011-11-26
comments: true
categories: emacs
---

I am in the process of rebooting my GTD workflow.  Since I do most of
my work in Emacs, I decided it was also time to refactor and improve my Emacs
configuration.  What was wrong with my configuration?

* I was living in the past (Emacs 23) and wanted to live in the future
  (Emacs 24).
* My configuration file was not under version control.  And, because
  of the differences between Fedora and OS X, the configuration on my
  work laptop and personal laptop had drifted significantly.
* My `.emacs` file was poorly structured and easily broken.

I've documented my work on my Emacs configuration here, mostly as a
means of reminding my future self what I did.  Overall the experience
was pleasant.  Emacs 24 brings many long-needed features to Emacs.

The end result can be seen below.

{% img /assets/emacs-desktop.png %}


# Installing Emacs 24:

On Fedora, I installed directly from git:
``` bash
# Install Dependencies
sudo yum-builddep emacs
sudo yum install
# Get source
cd ~/src
git clone git://git.savannah.gnu.org/emacs.git
cd emacs
# Install
./autogen.sh
./configure
make
sudo make install
```

On my Mac, I installed Emacs 24 via Homebrew:
``` bash
brew uninstall emacs
brew install emacs --cocoa --use-git-head --HEAD
ln -s /usr/local/Cellar/emacs/HEAD/Emacs.app /Applications/
```

# Rebuilding My Emacs Configuration

A number of high-quality, pre-built Emacs configurations are available
online, including:

* [Emacs Starter Kit](https://github.com/technomancy/emacs-starter-kit)
* [Emacs Prelude](https://github.com/bbatsov/emacs-prelude)
* [Emacs Dev Kit](https://github.com/bbatsov/emacs-dev-kit)

A few Opscoders use configurations based around Emacs Starter
Kit. However, since I have fairly simple configuration needs, I decided
to borrow a few ideas from each and forge ahead on my own.  The result
can be found in my
[emacs-config](https://github.com/stevendanna/emacs-config)
repository.  It is still very much a work in progress.

I've structured my new `.emacs.d` directory as follows:

    .
    |-- init.el
    |-- modules/
    |-- snippets/
    |-- themes/
    `-- vendor/

* `init.el`: The main entry point of the Emacs configuration.  I've
  taken a bit of code from the README of Emacs Starter Kit that
  automatically installs packages from ELPA.

* `modules/`: This is the heart of my emacs config.  Any `.el` files
  in this directory are automatically loaded.

* `snippets/`: Used to store snippets for yasnippet.  Currently empty.

* `themes/`: Color themes.

* `vendor/`: Any 3rd party Emacs packages that are not yet distributed
  via ELPA.

The auto-installation of packages is great for sharing my
configuration between workstations; but, it can be brutally slow the
first time you run it.

## Highlights

The majority of my emacs configuration consists of minor
customizations to popular, pre-built packages. The following are the
packages that I use on a daily basis:

* [ido-mode](http://www.emacswiki.org/emacs/InteractivelyDoThings): I
   honestly don't know how I used Emacs before discovering ido-mode.

* [org-mode](http://orgmode.org/): Org-mode is my go-to application
  anytime I need to take notes in a meeting, outline a new project, or
  author a simple document.

* [ess](http://ess.r-project.org/): Emacs Speaks Statistics (ESS) is a
   must-have for any user of R and Emacs. Kieran Healy has created
   [a repository for ESS](http://kieranhealy.org/packages/), making
   ESS easily installable via Emacs 24's package management features.

* [auctex](http://www.gnu.org/s/auctex/): AucTeX provides a number a
  features useful to those who love LaTeX.  If you haven't fallen in
  love with LaTeX yet, I highly recommend trying it!

In addition to the tried-and-true packages above and a handful of
popular programming modes, I added the following packages to my
standard configuration:

* [Gist](https://github.com/defunkt/gist.el): I use Github's gists all
  the time; I can't believe I didn't look for this before.

* [Deft](http://jblevins.org/projects/deft/): Deft allows you quickly
  create, edit, and view plain-text notes.  I've configured it to use
  org-mode as its default text-mode and hope to use it to easily
  collect tasks throughout the day.

* [hippie-expand](http://www.emacswiki.org/emacs/HippieExpand): I
  previously used predictive-mode extensively, but eventually found it
  too slow to use productively.  So far, hippie-expand has been
  filling the auto-completion void that I've been feeling since remove
  predictive-mode.

All of these packages are either shipped with Emacs or available as
packages using the new package management features in Emacs 24.  I was
initially skeptical of the package management features.  However, I've
found that a large amount of the complexity in managing my previous
configuration was the result of manually managing packages.  With this
complexity removed, moving to Emacs 24 was relatively painless.

I was also pleased to see improvements to clipboard integration. My
old `.emacs` file had a number of ugly workarounds for properly
interacting with the clipboard on multiple platforms.  I've found this
completely unnecessary in Emacs 24.

The only problem I encountered during this transition was finding a
suitable color theme.  Low-contrast color themes such as zenburn seem
to be in vogue at the moment.  I prefer higher-contrast themes.  In
Emacs 23, I used dark-laptop, but it appears that this theme has not
yet been ported to Emacs 24's built-in color theme support.  My
current theme is a small color theme based heavily on dark-laptop.

## Deploy!

I've deployed my new configuration to all of my workstations via a
simple `git pull`.  My hope is that a few weeks of real-world use will
find the rough edges that remain.  Comments and suggestions welcome!
