---
layout: post
title: SSH Authentication with Chef
date: '2013-01-02'
description: Chef's documentation describes how to wrap SSH instead of configuring it. Here's how to configure it.

categories: ['chef', 'ssh']
versions:
    - name: SSH
      version: '5.9'
    - name: Chef
      version: 10.16.4
    - name: SSH-Util
      version: 0.6.0
---

## SSH known_hosts and Chef

A problem that the documentation for Chef's [Deploy_resource](http://wiki.opscode.com/display/chef/Deploy+Resource) talks about is a Chef run pausing while a program it runs waits for user input. One way this presents itself is with SSH's host fingerprint checking, which ensures that the host you're connecting to now is the same host you connected to earlier.  When first connecting to a host with anything that runs over SSH, you'll see something like this:

    root@precise64:/tmp# git clone git@github.com:rails/rails.git
    Cloning into 'rails'...
    The authenticity of host 'github.com (207.97.227.239)' can't be established.
    RSA key fingerprint is 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48.
    Are you sure you want to continue connecting (yes/no)?

Chef can't type 'yes' for this, and so it waits. And waits. The second issue that can come up is having a keyphrase on the SSH key. Normally, this is a very good idea, but it just doesn't work well with automated deployments because, again, it prompts for input.

To get around this, The Chef docs recommend creating a wrapper around SSH called `wrap-ssh4git.sh` which disables Strict Host Checking and tells SSH to use a key that has had it's keyphrase removed. While there's no *good* way to use a keyphrase protected key, we can use SSH's built in tools to add hosts to the `known_hosts` file that SSH consults when connecting.
<!-- more -->
## Adding to known_hosts

SSH includes a program called [ssh-keyscan](http://linux.die.net/man/1/ssh-keyscan) that we can use to add entries to our `known_hosts` file.

    root@precise64:/tmp# ssh-keyscan -H github.com
    # github.com SSH-2.0-OpenSSH_5.5p1 Debian-6+squeeze1+github8
    |1|7+BJH9KqHv6AMzOna9ewgQjgdP0=|lpk0k7TfHbt/QTIbvX28s61Bm8I= ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAq2A7hRGmdnm9tUDbO9IDSwBK6TbQa+PXYPCPy6rbTrTtw7PHkccKrpp0yVhp5HdEIcKr6pLlVDBfOLX9QUsyCOV0wzfjIJNlGEYsdlLJizHhbn2mUjvSAHQqZETYP81eFzLQNnPHt4EVVUh7VfDESU84KezmD5QlWpXLmvU31/yMf+Se8xhHTvKSCZIFImWwoG6mbUoWf9nzpIoaSjB+weqqUUmpaaasXVal72J+UX2B+2RPW3RcT0eOzQgqlJL3RKrTJvdsjE3JEAvGq3lGHSZXy28G3skua2SmVi/w4yCE6gbODqnTWlg7+wC604ydGXA8VJiS5ap43JXiUFFAaQ==

 Recent versions of SSH hash the hostname in `known_hosts`, which makes it harder for an intruder to get a list of hosts that the system presumably has access to, but it also makes it harder for us to find where [github.com](http://github.com) is in that list. To see if your system hashes the `known_hosts` file, open up the system's `ssh_config` file (mine was at `/etc/ssh/ssh_config`) and look for `HashKnownHosts yes`. That `-H` flag on `ssh-keyscan` toggles whether to hash the output, and then we can dump it into the `known_files`

    ssh-keygen -H -F github.com >> ~/.ssh/known_hosts

## Chef-SSH Cookbook

To make this process simpler for myself, I wrote a [Chef cookbook](https://github.com/markolson/chef-ssh) that adds a feature letting you do the following:

    ssh_known_hosts "github.com" do
      hashed true
      user 'webapp'
    end

You can install it simply by running `knife cookbook download ssh-util`, and the [README](https://github.com/markolson/chef-ssh/blob/master/README.md) has more details about the options available.

## SSH Config

I use seperate keys to authenticate against different services, including a unique (keyphraseless) key just for github. SSH reads from configuration files before connecting, which lets you specify seperate options for different hosts. The per-user file is normally kept at  '~/.ssh/config', and documentation for [SSH Config](http://www.openbsd.org/cgi-bin/man.cgi?query=ssh_config) could be better, in practice they're simple enough

    Host github.com
      User git
      IdentityFile ~/.ssh/github.pem

Anything connects over SSH will now check if it's connecting to 'github.com', and if so, will use the file `~/.ssh/github.pem` instead of the default `~/.ssh/id_rsa`. Included in the [cookbook](https://github.com/markolson/chef-ssh) is a resource to manage options in `ssh_config` files.

    ssh_config "github.com" do
      options 'User' => 'git', 'IdentityFile' => '~/.ssh/github.pem'
      user 'webapp'
    end

## Deploy Revision

Presuming that the `github.pem` file exists on the system, and with github added to our `known_hosts`, plus the entry `ssh_config` file, Chef can run deploy revision as `webapp` without any interruptions.

    deploy "private_repo" do
      repo "git@github.com:acctname/private-repo.git"
      user "webapp"
      deploy_to "/tmp/private_code"
      action :deploy
    end

