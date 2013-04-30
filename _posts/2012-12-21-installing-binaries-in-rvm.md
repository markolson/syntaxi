---
layout: post
title: Installing Prebuilt Binaries with RVM
date: '2012-12-21'
description: Building, packaging, and re-using prebuilt rubies in RVM isn't well documented, but is really really useful.
categories: ['rvm']
versions:
    - name: Ubuntu
      version: '12.04'
    - name: RVM
      version: 1.17.10
---

I've been working with a lot of virtual machines lately, and using RVM to install the version of ruby I need took longer than installing the base system. If you go through the motions of `rvm install {VERSION}`, you might just gloss over this message:

```text
No binary rubies available for: ubuntu/12.04/x86_64/{VERSION}.<br />
Continuing with compilation. Please read 'rvm mount' to get more information on binary rubies.
```

Not reading the documentation for `rvm mount` might end up costing you a lot of time.

<!-- more -->

## RVM Prepare

RVM has a list of premade binaries that it will download if it gets the chance - you can see the list in `/usr/local/rvm/config/remote` or [on github](https://github.com/wayneeseguin/rvm/blob/master/config/remote) (I used a system-wide RVM install for this. Your path may be ~/.rvm/config/remote). For binaries that don't exist in that list, there's the option of packaging your own.

I'll be assuming that we're building a 2.0.0-preview2 binary, using a server you can scp files to, which is also accessable at `http://artifacts.corp/binaries/`

```bash
    rvm install 2.0.0-preview2
    rvm prepare 2.0.0-preview2 --install -r artifacts.corp:/var/www/binaries/
```

The `rvm prepare` command should have made a file called `ruby-2.0.0-preview2.tar.bz2` in your current directory, and output something like the following.

```text
--- upload:
ssh "artifacts.corp" "mkdir -p /var/www/binaries//ubuntu/12.04/x86_64/"
scp "ruby-2.0.0-preview2.tar.bz2" "artifacts.corp:/var/www/binaries//ubuntu/12.04/x86_64/ruby-2.0.0-preview2.tar.bz2"
```

RVM rightly cares quite a bit about the OS that the build will be installed on, and **strongly** recommends that you follow the naming convention seen in that output as a way to organize and name your builds. Once those commands are run and the file is uploaded, we'll be ready for  the next time that we need to install 2.0.0-preview2 on this OS.

##RVM Mount

There are two ways to download premade binaries - `rvm install {name}`, which uses the list of prebuilt binaries if possible, or `rvm mount -r {URI}`. The former works with data from configuration files, while the latter works using just command line parameters. Let's install our Ruby using mount first, since it takes fewer steps:

```bash
rvm mount -r http://artifacts.corp/binaries//ubuntu/12.04/x86_64/ruby-2.0.0-preview2.tar.bz2 --verify-downloads 1
```

On the VM I tested this with, `rvm mount` takes 12 seconds, while `rvm install` takes 4 minutes. Of course, there's that `verify-downloads` flag and the lists of prebuilt binaries left to explain..

##RVM Remote

As mentioned above, RVM has a list of prebuilt binaries for different platforms that it can download. In addition to the file (on ubuntu) at `/usr/local/rvm/config/remote`, RVM will also use `/usr/local/rvm/users/remote`, if present. You can see the rubies that RVM can download as binaries for your system with `rvm list remote`
since it takes fewer steps:

```bash
# Rubies available for 'ubuntu/12.04/x86_64':

    rbx-2.0.0-rc1
    ruby-1.9.3-p194
    ruby-1.9.3-p286
    ruby-1.9.3-p327
```

If we add the URL to our precompiled 2.0.0-preview2 binary to `/usr/local/rvm/users/remote`, it will show up in that list.
since it takes fewer steps:

```bash
echo "http://artifacts.corp/binaries//ubuntu/12.04/x86_64/ruby-2.0.0-preview2.tar.bz2" >> /usr/local/rvm/user/remote
rvm list remote
```
```text
# Rubies available for 'ubuntu/12.04/x86_64':

    rbx-2.0.0-rc1
    ruby-1.9.3-p194
    ruby-1.9.3-p286
    ruby-1.9.3-p327
    ruby-2.0.0-preview2
```

**For this bit, there is every chance that I am missing, or misunderstanding, something. This may not be the best or right way to do things**

Even though it shows up in the remote list, we still have to tell RVM that our server is an acceptable place to download rubies from. We do this by adding it to another textfile.

```bash
    echo "rvm_remote_server_url=http://artifacts.corp/" >> /usr/local/rvm/user/db
```

## Checksums

At this point, if you try `rvm install 2.0.0-preview2` you would get an error message about checksums not matchng. With RVM's mount we used the `--verify-downloads` flag, which saves checksums of the file we download after the fact. Using that same flag, `rvm install 2.0.0-preview2 --verify-downloads 1`, will download and install the package correctly.

If however, you want to have RVM to use checksums to verify the download was successful, we can refer way back to the rest of the output from our `rvm prepare` command:

```text
    --- rvm/config/md5:
    http://artifacts.corp/binaries/ruby-2.0.0-preview2.tar.bz2=fdb22cbad861616f5e3b56f0e3d976be
    --- rvm/config/sha512:
    http://artifacts.corp/binaries/ruby-2.0.0-preview2.tar.bz2=eb1972575cee09b0de59f39815b2f9992366cd6aaf3e32ab214d39b054029cf904260933e8b2fa101c7b5eb548d013e0e05c09d3e93dbc97a1ae55789d6a046b
```

And add those lines to our `user/*` files

```bash
    echo "http://artifacts.corp/binaries/ubuntu/12.04/x86_64/ruby-2.0.0-preview2.tar.bz2=eb1972575cee09b0de59f39815b2f9992366cd6aaf3e32ab214d39b054029cf904260933e8b2fa101c7b5eb548d013e0e05c09d3e93dbc97a1ae55789d6a046b" >> /usr/local/rvm/user/sha512
    echo "http://artifacts.corp/binaries/ubuntu/12.04/x86_64/ruby-2.0.0-preview2.tar.bz2=fdb22cbad861616f5e3b56f0e3d976be" >> /usr/local/rvm/user/md5
```

Now, `rvm install 2.0.0-preview2` will download a prebuilt binary from a server of your choosing, as well as verify that the checksum is the same as the package you built at the very beginning.

Phew.


