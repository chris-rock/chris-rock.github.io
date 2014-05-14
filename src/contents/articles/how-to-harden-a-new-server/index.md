---
title: How to harden a new server with Chef
author: chris
date: 2014-05-14
template: article.jade
---

Deutsche Telekom developed scripts in Chef and Puppet to harden servers according to well-known guidelines like bettercrypto and internal guidelines at Deutsche Telekom.

Disclosure: I am core developer at this project.

Today we cook with [knife-solo](http://matschaffer.github.io/knife-solo/) and harden a fresh copy of Ubuntu 14.04. The process of hardening a server is quite difficult and takes a lot of knowledge and experience. Even the most-experienced administrators rely on guidelines to meet the best practices.

Since we live in a cloud world by now, [Dominik Richter](http://arlimus.github.io/), Patrick Meier and myself invented the `Security Automation Toolkit`. It is a set of cookbooks that help you to secure your server with well-known rules. 

The following steps illustrate the hardening process of a fresh server with nothing than a base Linux system.

I assume you have [Chef Development Kit](http://www.getchef.com/downloads/chef-dk/mac/) or [Chef](http://www.getchef.com/chef/install/) and [Berkshelf](http://berkshelf.com/) on your machine.

At first of all we need to install `knife-solo` via `gem` on your workstation.


```bash
∅> sudo gem install knife-solo
Fetching: method_source-0.8.2.gem (100%)
Successfully installed method_source-0.8.2
Fetching: slop-3.5.0.gem (100%)
Successfully installed slop-3.5.0
Fetching: coderay-1.1.0.gem (100%)
Successfully installed coderay-1.1.0
Fetching: pry-0.10.0.pre2.gem (100%)
Successfully installed pry-0.10.0.pre2
Fetching: chef-zero-2.0.2.gem (100%)
Successfully installed chef-zero-2.0.2
Fetching: diff-lcs-1.2.5.gem (100%)
Successfully installed diff-lcs-1.2.5
Fetching: highline-1.6.21.gem (100%)
Successfully installed highline-1.6.21
Fetching: net-ssh-gateway-1.2.0.gem (100%)
Successfully installed net-ssh-gateway-1.2.0
Fetching: net-ssh-multi-1.2.0.gem (100%)
Successfully installed net-ssh-multi-1.2.0
Fetching: mime-types-1.25.1.gem (100%)
Successfully installed mime-types-1.25.1
Fetching: rest-client-1.6.8.rc1.gem (100%)
Successfully installed rest-client-1.6.8.rc1
Fetching: ipaddress-0.8.0.gem (100%)
Successfully installed ipaddress-0.8.0
Fetching: mixlib-shellout-1.4.0.gem (100%)
Successfully installed mixlib-shellout-1.4.0
Fetching: mixlib-config-2.1.0.gem (100%)
Successfully installed mixlib-config-2.1.0
Fetching: mixlib-cli-1.5.0.gem (100%)
Successfully installed mixlib-cli-1.5.0
Fetching: systemu-2.5.2.gem (100%)
Successfully installed systemu-2.5.2
Fetching: ohai-7.0.4.gem (100%)
Successfully installed ohai-7.0.4
Fetching: chef-11.14.0.alpha.2.gem (100%)
Successfully installed chef-11.14.0.alpha.2
Fetching: knife-solo-0.4.1.gem (100%)
Thanks for installing knife-solo!

If you run into any issues please let us know at:
  https://github.com/matschaffer/knife-solo/issues

If you are upgrading knife-solo please uninstall any old versions by
running `gem clean knife-solo` to avoid any errors.

See http://bit.ly/CHEF-3255 for more information on the knife bug
that causes this.
Successfully installed knife-solo-0.4.1
Installing ri documentation for chef-11.14.0.alpha.2
Installing ri documentation for chef-zero-2.0.2
invalid options: -SNw2
(invalid options are ignored)
Installing ri documentation for coderay-1.1.0
Installing ri documentation for diff-lcs-1.2.5
Installing ri documentation for highline-1.6.21
Installing ri documentation for ipaddress-0.8.0
Installing ri documentation for knife-solo-0.4.1
Installing ri documentation for method_source-0.8.2
Installing ri documentation for mime-types-1.25.1
Installing ri documentation for mixlib-cli-1.5.0
Installing ri documentation for mixlib-config-2.1.0
Installing ri documentation for mixlib-shellout-1.4.0
Installing ri documentation for net-ssh-gateway-1.2.0
Installing ri documentation for net-ssh-multi-1.2.0
Installing ri documentation for ohai-7.0.4
Installing ri documentation for pry-0.10.0.pre2
Installing ri documentation for rest-client-1.6.8.rc1
Installing ri documentation for slop-3.5.0
Installing ri documentation for systemu-2.5.2
19 gems installed
∅> 
```

As a next step we need to download the `hardening-kitchen`. This comes with a set of hardening cookbooks and works out of the box. You're free to extend the kitchen with your own cookbooks.

```bash
git clone https://github.com/TelekomLabs/example-chef-hardening.git
cd example-chef-hardening
```

By now, the workstation is ready and we are able to bootstrap our server with Chef and our hardening cookbooks.

```bash
knife solo bootstrap ubuntu@hardening.tlabscloud.com nodes/default.json 
Bootstrapping Chef...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 15934  100 15934    0     0  21847      0 --:--:-- --:--:-- --:--:-- 21857
Downloading Chef 11.10.4 for ubuntu...
downloading https://www.opscode.com/chef/metadata?v=11.10.4&prerelease=false&nightlies=false&p=ubuntu&pv=14.04&m=x86_64
  to file /tmp/install.sh.30630/metadata.txt
trying wget...
url https://opscode-omnibus-packages.s3.amazonaws.com/ubuntu/13.04/x86_64/chef_11.10.4-1.ubuntu.13.04_amd64.deb
md5 25e27eeaff3304df354e735a56005f04
sha256  98fc3ef27419ec45281306b160e5f1e736a9846ad9060a151046d5a29c49af6b
yolo    true
downloaded metadata file looks valid...
downloading https://opscode-omnibus-packages.s3.amazonaws.com/ubuntu/13.04/x86_64/chef_11.10.4-1.ubuntu.13.04_amd64.deb
  to file /tmp/install.sh.30630/chef_11.10.4-1.ubuntu.13.04_amd64.deb
trying wget...
Comparing checksum with sha256sum...

WARNING: Chef-Client has not been regression tested on this O/S Distribution
WARNING: Do not use this configuration for Production Applications.  Use at your own risk.

Installing Chef 11.10.4
installing with dpkg...
dpkg: warning: downgrading chef from 11.12.4-1 to 11.10.4-1.ubuntu.13.04
(Reading database ... 62386 files and directories currently installed.)
Preparing to unpack .../chef_11.10.4-1.ubuntu.13.04_amd64.deb ...
Unpacking chef (11.10.4-1.ubuntu.13.04) over (11.12.4-1) ...
Setting up chef (11.10.4-1.ubuntu.13.04) ...
Thank you for installing Chef!
Running Chef on hardening.tlabscloud.com...
Installing Berkshelf cookbooks to 'cookbooks'...
Installing os-hardening (1.0.1) from site: 'http://cookbooks.opscode.com/api/v1/cookbooks'
Installing ssh-hardening (1.0.0) from site: 'http://cookbooks.opscode.com/api/v1/cookbooks'
Using sysctl (0.3.4)
Using ntp (1.6.2)
Using apt (2.3.8)
Using yum (2.4.4)
Uploading the kitchen...
Generating solo config...
Running Chef...
Starting Chef Client, version 11.10.4
[2014-05-14T14:53:23+00:00] WARN: unable to detect ip6address
Compiling Cookbooks...
[2014-05-14T14:53:23+00:00] WARN: encrypted_data_bag_Secret is set but file does not exist. Unsetting
/home/ubuntu/chef-solo/cookbooks-2/chef-solo-search/libraries/search.rb:57: warning: already initialized constant PARSER_LOADED

...

Chef Client finished, 10/45 resources updated in 4.206405118 seconds

```

In less than 5 minutes, a new server is hardened for the core system and ssh. Be aware that your server still needs some configuration in order to be production ready. For that I recommend to:

 * Configure  your firewall properly
 * Install  critical patches
 * Use patch management
 * Use system monitoring

More information is available at:

 * http://telekomlabs.github.io/
 * https://github.com/TelekomLabs/example-chef-hardening
 * http://community.opscode.com/cookbooks/os-hardening
 * http://community.opscode.com/cookbooks/ssh-hardening

Happy Hardening!

