---
layout: post
title:  "Install Jekyll for Github Pages"
date:   2023-05-11
categories: github pages jekyll install
---

This describes how to install [Ruby](https://www.ruby-lang.org) and
[Jekyll](https://jekyllrb.com) on MacOS Ventura so that you can get
started with [GitHub
Pages](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll).

It assumes you have Xcode installed, including Xcode command line
tools.

# Don't use your system ruby.

You need ```ruby```, ```bundle```, and ```gem```. You'll find that
MacOS comes with all three:

```
$ which ruby bundle gem
/usr/bin/ruby
/usr/bin/bundle
/usr/bin/gem
```

**Do not use these.**

They are there for the OS to use for its own purposes. The version of
ruby included with the OS may not be that required by [Github
Pages](https://pages.github.com/versions) (indeed, is not on MacOS
13.3.1). More importantly, if you attempt to install jekyll using the
system's gem you will be greeted with:

```
$ gem install jekyll
ERROR:  While executing gem ... (Gem::FilePermissionError)
    You don't have write permissions for the /Library/Ruby/Gems/2.6.0 directory.
```

This error is reminding you that you are using the system's ruby
installation. Don't be tempted to use sudo to work around the
permissions error. There's a better way.

# Install ruby using the rvm package manager.

The easiest way to setup a ruby environment is to use a package
manager of some sort to install a local version of ruby that is
dedicated to development.  There are many ways to do this. Here's how
to do it with [rvm](https://rvm.io/) .

First, you'll need [GnuPG](https://gnupg.org/). It's probably easiest
to get that from [port](https://www.macports.org) or
[brew](https://brew.sh/). I used port:

```
$ sudo port install gnupg2
```

Now follow the instructions at [rvm](https://rvm.io) instructions to
get started with rvm itself:

```
gpg2 --keyserver keyserver.ubuntu.com --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
curl -sSL https://get.rvm.io | bash -s stable
```

When the rvm install finishes it will ask to setup your environment:

```
source "$HOME/.rvm/scripts/rvm"
```

Now you have the rvm command in your path:

```
$ which rvm
~/.rvm/bin/rvm
```

Next use rvm to install ruby into your local environment (i.e. into
~/.rvm).

[Githug Pages depends](https://pages.github.com/versions/) on ruby
version 2.7. It should be as simple to install as ```rvm install
2.7```. However, if you try that, you may discover that the rvm ruby
build fails because it has a dependency on openssl version 1.1, and
you may not have openssl 1.1 installed on your system, or if it is
installed then the rvm builder doesn't know where.

Rather than struggling with openssl dependencies and whatever version
of opensll you happen to have installed elsewhere on your system, use
rvm to install the correct openssl version in your rvm environment:

```
rvm pkg install openssl
```

Now build ruby as follows:

```
PKG_CONFIG_PATH="$HOME/.rvm/usr/lib/pkgconfig" rvm install 2.7 --with-openssl-dir="$HOME/.rvm/usr"
```

You should now have local versions of ruby, bundle, and gem:

```
$ which ruby bundle gem
~/.rvm/rubies/ruby-2.7.2/bin/ruby
~/.rvm/rubies/ruby-2.7.2/bin/bundle
~/.rvm/rubies/ruby-2.7.2/bin/gem
```

Finally, install jekyll:

```
gem install jekyll
```

And with that, you will have satisfied the GitHub Pages Jekyll
prerequisites.

# Install script

Here is everything in one script that assumes gpg2 is available.

```
#! /bin/bash
set -e
gpg2 --keyserver keyserver.ubuntu.com --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
curl -sSL https://get.rvm.io | bash -s stable
source "$HOME/.rvm/scripts/rvm"
rvm pkg install openssl
PKG_CONFIG_PATH="$HOME/.rvm/usr/lib/pkgconfig" rvm install 2.7 --with-openssl-dir="$HOME/.rvm/usr"
gem install jekyll
```

# Addendum

Note: There is a long thread on Stackoverflow where people discuss
their [troubles with rvm and
openssl](https://stackoverflow.com/questions/15511943/troubles-with-rvm-and-openssl).

Note: The `rvm pkg ...` command is [deprecated](https://rvm.io/packages) at
the time of this writing.