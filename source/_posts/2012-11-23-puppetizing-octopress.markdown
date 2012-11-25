---
layout: post
title: "Puppetizing Octopress"
date: 2012-11-23 12:29
comments: true
categories: 
---

## So Stuffed, Must Automate

For #hacksgiving this year, I decided I'd finally make use of my year old puppetize.it domain name and start a blog on building infrastructure with Puppet and Puppet Forge modules.

I'm the product owner for Puppet Forge at Puppet Labs. I got my start at the company in professional services and came from Penn State before that where I helped one of the central IT teams run linux infrastructure for thousands of faculty, staff and students.

The [Puppet Forge](http://forge.puppetlabs.com) is home to over 600 modules (pre-written Puppet code) for automating all kinds of things. When you see someone demo Puppet, oftentimes they'll use a module to build a brand new Wordpress instance. It demonstrates one of the benefits of Puppet but Wordpress isn't that compelling to me as a blogging platform.

This #hacksgiving, I decided to give [Octopress](http://octopress.org/) a try. Octopress is a static-site blogging framework that lets me write my posts in markdown and comes with all the styling work done for me. You can host your blog easily on GitHub pages or Heroku but for the sake of playing around, I decided to host mine in a micro EC2 instance.

## The Ingredients

* [Free License of Puppet Enterprise](http://puppetlabs.com/puppet/puppet-enterprise/)  
PE is free for up to 10 machines, more than enough for my needs. It handles the installation of a scalable Puppet Master, a graphical console and MCollective. It also comes with a provisioner for AWS machines which will be handy when I go to deploy this blog.
* [Modules from the Puppet Forge]()  
I don't want to re-invent the wheel and don't want you to either. So, I turn to the Puppet Forge to find almost everything I need. 
  * [puppetlabs/vcsrepo](https://forge.puppetlabs.com/puppetlabs/vcsrepo) lets me do a git clone of the Octopress repository.  
  * [alup/rbenv](https://forge.puppetlabs.com/alup/rbenv) lets me easily provide Ruby 1.9.3 on my RedHat 6 machine.
  * [rcoleman/octopress](https://forge.puppetlabs.com/rcoleman/octopress) is a module I whipped up that declares resources from the two modules above, specifically for bootstrapping my Octopress blog. I'll probably expand its functionality at a later date but right now it's just a convenience module.    
  * [puppetlabs/apache](http://forge.puppetlabs.com/puppetlabs/apache) makes it dead-simple to host my blog via the httpd web server.
* [Linux node in Amazon Web Services](http://aws.amazon.com/)

## RColeman/Octopress

To make things a little more modular, I decided to whip up an Octopress module for the Forge that is responsible for creating an octopress user & group, installs Ruby 1.9.3 via RBENV and clones my Octopress repository. The Octopress class will be responsible for updating that repository with new posts as they're made.

## Baking the Blog

To kick things off, I used the `puppet node_aws` command provided by Puppet Enterprise to get myself a micro instance in the us-west-2 region of Amazon Web Services. Using `puppet node_aws` is actually pretty complicated so I won't go into detail here. If you're interested, comment on the post and perhaps I'll follow-up in more detail. Otherwise, you basically need to follow the [documentation here](http://docs.puppetlabs.com/guides/cloud_pack_getting_started.html).

I'm being a bit of a cheapskate, forgoing the bells and whistle of a full fledged PE Master for the tiny memory footprint of a master-less `puppet apply` setup.

The installation of Puppet Enterprise is easy and boring so I'll skip that portion and [refer you to the documentation](http://docs.puppetlabs.com/pe/2.7/), continuing on to create a directory for my modules and install some via the `puppet module` tool.

{% codeblock %}
mkdir /etc/puppetlabs/puppet/modules
puppet module install puppetlabs-vcsrepo
puppet module install alup-rbenv
puppet module install rcoleman-octopress
puppet module install puppetlabs-apache
{% endcodeblock %}

With this content installed, its time to write my main manifest which I'll run `puppet apply` against at regular intervals.

{% codeblock octopress.rb (really .pp, but for syntax highlighting) %}
# This class sets everything up, based on my own fork of Octopress
class { 'octopress':
  source_code => 'git://github.com/rcoleman/octopress.git',
}

# I want Puppet to run every 10 minutes against this manifest
cron { 'run_puppet':
    command => 'FACTER_osfamily=Redhat /opt/puppet/bin/puppet apply /root/octopress.pp',
    user    => root,
    minute  => '*/10'
}

# I want to host the blog through Apache
include apache
apache::vhost { 'puppetize.it':
    priority        => '10',
    vhost_name      => 'puppetize.it',
    port            => '80',
    docroot         => '/opt/octopress/public',
    serveradmin     => 'ryancoleman@me.com',
    serveraliases   => ['puppetize.it'],
}

{% endcodeblock %}

That's really all there is to this. The octopress class abstracts away a few resource declarations but this deployment is fairly simple. If I wanted to, I could create my own site-specific module with a class that contains these resources but for my purposes, that's overkill.


With one run of `puppet apply` against the above manifest, everything kicks off, including my cron to run puppet from here on out. I needed to override Facter on my Amazon Redhat instance that reports Linux as its osfamily because alup-rbenv doesn't explicitly support Amazon's flavor of Linux. There are more flavors of Linux than of gravy, I'm certain of it.

{% codeblock puppetrun.sh %}
[root~]# FACTER_osfamily=Redhat puppet apply octopress.pp
notice: /Stage[main]/Rbenv::Dependencies::Centos/Package[libtool]/ensure: created
notice: /Stage[main]/Rbenv::Dependencies::Centos/Package[make]/ensure: created
notice: /Stage[main]/Rbenv::Dependencies::Centos/Package[git]/ensure: created
notice: /Stage[main]/Rbenv::Dependencies::Centos/Package[gcc-c++]/ensure: created
notice: /Stage[main]/Rbenv::Dependencies::Centos/Package[openssl-devel]/ensure: created
notice: /Stage[main]//Cron[run_puppet]/ensure: created
notice: /Stage[main]/Rbenv::Dependencies::Centos/Package[flex]/ensure: created
notice: /Stage[main]/Rbenv::Dependencies::Centos/Package[patch]/ensure: created
notice: /Stage[main]/Rbenv::Dependencies::Centos/Package[bison]/ensure: created
notice: /Stage[main]/Rbenv::Dependencies::Centos/Package[readline-devel]/ensure: created
notice: /Stage[main]/Octopress/Group[octopress]/ensure: created
notice: /Stage[main]/Octopress/Vcsrepo[/opt/octopress]/ensure: Creating repository from present
notice: /Stage[main]/Octopress/Vcsrepo[/opt/octopress]/ensure: created
notice: /Stage[main]/Octopress/User[octopress]/ensure: created
notice: /Stage[main]/Octopress/File[/opt/octopress]/owner: owner changed 'root' to 'octopress'
notice: /Stage[main]/Octopress/File[/opt/octopress]/group: group changed 'root' to 'octopress'
notice: /Stage[main]/Octopress/Exec[bundle install]/returns: executed successfully
notice: /Stage[main]/Rbenv::Dependencies::Centos/Package[gettext]/ensure: created
notice: /Stage[main]/Octopress/Rbenv::Install[octopress]/Exec[rbenv::checkout octopress]/returns: executed successfully
notice: /Stage[main]/Octopress/Rbenv::Compile[1.9.3-p327]/Rbenv::Plugin::Rubybuild[rbenv::rubybuild::octopress]/Rbenv::Plugin[rbenv::plugin::rubybuild::octopress]/File[rbenv::plugins octopress]/ensure: created
notice: /Stage[main]/Octopress/Rbenv::Compile[1.9.3-p327]/Rbenv::Plugin::Rubybuild[rbenv::rubybuild::octopress]/Rbenv::Plugin[rbenv::plugin::rubybuild::octopress]/Exec[rbenv::plugin::checkout octopress ruby-build]/returns: executed successfully
notice: /Stage[main]/Octopress/Rbenv::Install[octopress]/File[rbenv::rbenvrc octopress]/ensure: defined content as '{md5}51b44c6ccc11da8b43b52849082e4ffe'
notice: /Stage[main]/Octopress/Rbenv::Install[octopress]/Exec[rbenv::shrc octopress]/returns: executed successfully
notice: /Stage[main]/Octopress/Rbenv::Compile[1.9.3-p327]/Exec[rbenv::compile octopress 1.9.3-p327]/returns: executed successfully
notice: /Stage[main]/Octopress/Rbenv::Compile[1.9.3-p327]/Rbenv::Gem[rbenv::bundler octopress 1.9.3-p327]/Rbenvgem[octopress/1.9.3-p327/bundler/present]/ensure: created
notice: /Stage[main]/Octopress/Rbenv::Compile[1.9.3-p327]/Exec[rbenv::rehash octopress 1.9.3-p327]/returns: executed successfully
notice: Finished catalog run in 1643.80 seconds
{% endcodeblock %}

The price of being cheap is a 1600 second catalog run, waiting for Ruby to compile. :-)

That's all folks! This wasn't the most glamorous of blog posts but it's my inaugural post and everything you read above is what I did for you to read this text. Please consider coming back after I've tinkered with something else and I hope you had a great holiday!
