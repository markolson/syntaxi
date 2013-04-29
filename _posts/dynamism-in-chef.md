---
title: Gems and Git with Chef
date: '2012-12-13'
description:
categories: 'coding'
tags: ['ruby', 'chef']

type: draft
---

## Starting with Chef

Where I work, we're using Chef to provision and update clusters of servers, but also to automate testing by installing our full stack of software on VMs, passing data through the whole backend pipeline. No mocks, stubs, or fixtures - we want raw data to go in, and results to show up in the frontend application.

Installing gems with Chef is pretty straightforward - you can just use their wrapper around the [Package resource](http://wiki.opscode.com/display/chef/Resources#Resources-Package).

*cookbooks/thisapp/recipes/default.rb*

	gem_package "thisgem" do
		version node['thisapp']['version']
		source "http://gems.corp"
	end


*cookbooks/thisapp/attributes/default.rb*

	default['thisapp']['version'] = '1.3.4'

### Adding Git

We get a lot of flexibility with this builtin resource, as we can build and test beta versions of gems on different environments by overriding the version attribute. I'm very, *very* lazy though, and I don't want to build and publish a gem each time I want to test it in a 'live' setting. Or perhaps I'm worried I might step on someone else's code if we're both working on '1.4.0beta'. I just want to push a branch of code to github, and test with that.

To get to the point where you (as root, for simplicity) can pull from github on a newly installed system, you have to do 3 things:

First: Add github's sitekey to the machine's *known_hosts* file

	execute "known_hosts for github" do
		command "ssh-keyscan -H github.com >> /root/.ssh/known_hosts"
		creates '/root/.ssh/known_hosts'
		user 'root'
		group 'root'
		umask '0600'
		action :run
	end

Second: Add a key that's authorized to access github to the system.

I originally did this by adding a keypair to the cookbook, but then starting putting it in a databag. <a href="#footnote1"><sup>1</sup></a> These commands assume your id_rsa is approved on github:

	knife data bag create keys
	ruby -rjson -e "f=File.open('/tmp/github.json','w');f.write({:id => :github, :id_rsa => File.read('$HOME/.ssh/id_rsa').to_s}.to_json);"
	knife data bag from file keys /tmp/github.json


Finally: in the cookbook, we pull and save that key.

	file "/root/.ssh/id_rsa" do
		user 'root'
		owner 'root'
		group 'root'
		mode "0600"
		action :create
		content data_bag_item("keys", "github")['id_rsa']
	end


### Our gem resource

Chef has a _git_ resource you can use to clone repositories, but Bundler is built for gems and can install from git, so let's use that. To start, we'll modify the attributes in an environment to reflect the functionality we want in the end: build a gem from a git branch.

*environments/mark-refactor.rb*

	default_attributes({
		'thisapp' => {
			'version' => {
				'git' => {
					'branch' => 'mark-refactor'
				}
			}
		}
	})

And change the default attribute file

*cookbooks/thisapp/attributes/default.rb*

	default['fusionengine']['version']['gem'] = '=1.3.4'

Admittedly, I don't know much more than what I consider a "tutorial following" level of knowledge about writing Chef providers and resources, but we can add the following files:

*cookbooks/thisapp/resources/gem.rb*

	def initialize(*args)
		super
		@action = :install
	end
	actions :install

	attribute :name, :kind_of => String, :name_attribute => true, :required => true
	attribute :source
	attribute :version, :required => true


*cookbooks/thisapp/providers/gem.rb*

	action :install do
		gem_name = new_resource.name

		gem_package "thisgem" do
			version new_resource.version['gem']['resource_version']
			source "http://gems.corp"
		end
	end

Resources are callable by appending the cookbook's name with the resource name, which means we'll be able to call *thisapp_gem* (instead of gem_package) in our recipe

*cookbooks/thisapp/recipes/default.rb*

	thisapp_gem "thisgem" do
		version node['thisapp']['version']
		source "http://gems.corp"
	end

### Including Bundler

We still have to add the Gemfile template that our *thisapp_gem* resource will use. To keep the number of variables down to only the gem's name, I'm hardcoding in *gems.corp* and the *USERNAME* for the git path, assuming that the github repo is named after the gem.

*cookbooks/thisapp/templates/Gemfile*

	source 'http://gems.corp'
	source :rubygems

	gem '<%=@gem_name-%>' <% if @version['gem'] && !@version['git'] %>, '=<%=@version['gem']-%>' <% elsif @version['git'] %>, :git => 'git@github.com:USERNAME/<%=@gem_name-%>.git' <% if @version['git']['branch'] %>, :branch => '<%= @version['git']['branch'] -%>'<% end %><% end %>


With our Gemfile template available, we can replace the call to *gem_package* in the resource with the following

	shell "Versioned bundle install" do
	  cwd         "/tmp"
	  code        %{bundle install}
	  action :nothing
	end

	template "Versioned Gemfile" do
	  source "Gemfile"
	  path "/tmp/Gemfile"
	  owner "root"
	  group "root"
	  mode 0755
	  variables({:version => version, :git_repo => git_repo, :gem_name => gem_name})
	  notifies :run, resources(:rvm_shell => "Versioned bundle install"), :immediately
	end


When this recipe is run so that the cookbook's defaults are used, /tmp/Gemfile will be generated as

	source 'http://gems.corp'
	source :rubygems

	gem 'thisapp', '=1.3.4'


And if run using the *mark-refactor* environment created above, the Gemfile would end up as

	source 'http://gems.corp'
	source :rubygems

	gem 'thisapp', :git => 'git@github.com:USERNAME/thisapp.git', :branch => 'mark-refactor'


Very nice - we've handed off all the work of downloading a gem - whether prebuilt or from source - and its dependencies to Bundler. There's still a problem though.. if you don't use Bundler to load this gem, it will be locked up in Bundler's cache. For gems expose binaries which run as services, we may just want the gem installed so its binaries are placed in a bin directory.

### Building the gem

**Warning: As it stands, this code does not work with gems that require gems from git repositories in their own Gemfiles.**

For this case, we need to put the code Bundler has into a directory and build it. Add the following to our gem provider after the template was written.

	if version['git']
	  directory '/tmp/vendor' do
	    recursive true
	    action :delete
	  end
	  shell "Versioned bundle pack" do
	    cwd		"/tmp"
	    code	%{bundle pack --all}
	  end
	  shell "Versioned non-bundler install" do
	    cwd		'/tmp/vendor/cache'
	    code 	%{cd #{path}* && gem build *.gemspec && gem install *.gem}
	  end
	end

Following the code - if we're using a git version, delete a /tmp/vendor/ directory if it exists. Next, run *bundle pack --all* in /tmp/, which creates a /tmp/vendor/cache/ directory. Finally, start in /tmp/vendor/cache, move to the directory that contains the gem we want to build, then build and install it as a gem. And yes, this is actually slightly more work than just using the [The *git* resource](http://wiki.opscode.com/display/chef/Resources#Resources-Git).

### Ok bye!

By using Bundler and a bootstrap Gemfile, we're able to cut down on conditionals to switch between  and the *gem_package* resource that we originally used. We also get the **big** win of not having to worry about sources anywhere besides the Gemfile template.

Internally, we use [chef-rvm](https://github.com/fnichol/chef-rvm) and have this provider in a cookbook we use for common code, so I don't have it available as a standalone cookbook yet, but I wanted to document what we did, hoping it can be useful to someone else in the interim.

<a name="footnote1">1</a>: Databags are nice ways to store data that doesn't really belong in environments or configuration files, or that you just plain don't want checked into version control. Now I know this *should* be an encrypted databag, and that even then you *should* be using a per-application deploy key, but it works.
