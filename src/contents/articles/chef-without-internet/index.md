---
title: Chef without an internet connection / Uninstall Chef with Chef
author: chris
date: 2014-08-24
template: article.jade
---

Recently I had a discussion with a DevOps team about an installation of Chef without an internet connection. A normal chef bootstrap fetches the chef binaries via "curl -L https://www.opscode.com/chef/install.sh | sudo bash". This will happen, even if you use a Chef Server. Therefore you would require a connection to download the Chef client binaries. 

This blog post demonstrates a chef run without an internet connection. Be aware, that we proof the basic setup only. Cookbooks may depend on external urls, but most of them allow attribute overrides to set custom urls.

## Install Chef without internet

Depending on your operating system, [Chef provides multiple platform dependent installer](http://www.getchef.com/chef/install/). E.g. for Ubuntu you could go with:

```
wget https://opscode-omnibus-packages.s3.amazonaws.com/ubuntu/13.04/x86_64/chef_11.12.8-2_amd64.deb
```

This enables you to store the binaries on a local server within your internal network. Now, you need to transfer the package on a fresh system and you can install it:

```bash

echo "---> Chef install status:"
dpkg-query -l chef

echo "---> Install Chef"
dpkg -i chef_11.12.8-2_amd64.deb

echo "---> Chef install status:"
dpkg-query -l chef

echo "---> Start chef run"
# we use chef zero here (-z), if you use a chef server, this would work, too
chef-client -z -o chef-purge

echo "---> Chef install status:"
dpkg-query -l chef
```

This proofs that we are able to bootstrap a machine with a local chef binary. It would be amazing, if Chef Inc could provide apt and yum repositories. This would allow us to work with the standard operating system setup and we would be able to re-use existing package mirrors.

# Uninstall Chef with Chef

My customers often asked for a method to do a one-time install of [DevSec Hardening Framework](http://dev-sec.io/). In such a case it would be great, if we could remove the chef installer after the machine bootstrap.

Now fun begins and we proof, that we are able to uninstall the chef binary with a chef cookbook:

```ruby

## Uninstall chef
package "chef" do
  action :purge
end
```

The console output looks like:
```bash
19:11:34 ∅> vagrant up
Bringing machine 'core' up with 'virtualbox' provider...
[core] VirtualBox VM is already running.
19:12:11 ∅> vagrant provision
[core] Running provisioner: shell...
[core] Running: inline script
---> Chef install status:
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                             Version                           Description
+++-================================-=================================-==============================================================================
un  chef                             <none>                            (no description available)
Selecting previously unselected package chef.
(Reading database ... 61149 files and directories currently installed.)
Unpacking chef (from .../chef_11.12.8-2_amd64.deb) ...
Setting up chef (11.12.8-2) ...
Thank you for installing Chef!
---> Chef install status:
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                             Version                           Description
+++-================================-=================================-==============================================================================
ii  chef                             11.12.8-2                         The full stack of chef
---> Start chef run
[2014-07-14T17:12:31+00:00] WARN: No config file found or specified on command line, using command line options.
[2014-07-14T17:12:31+00:00] INFO: Auto-discovered chef repository at /vagrant
[2014-07-14T17:12:31+00:00] INFO: Starting chef-zero on port 8889 with repository at repository at /vagrant
  One version per cookbook

[2014-07-14T17:12:31+00:00] INFO: Forking chef instance to converge...
[2014-07-14T17:12:31+00:00] WARN: 
* * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * 
SSL validation of HTTPS requests is disabled. HTTPS connections are still
encrypted, but chef is not able to detect forged replies or man in the middle
attacks.

To fix this issue add an entry like this to your configuration file:

``
  # Verify all HTTPS connections (recommended)
  ssl_verify_mode :verify_peer

  # OR, Verify only connections to chef-server
  verify_api_cert true
``

To check your SSL configuration, or troubleshoot errors, you can use the
`knife ssl check` command like so:

``
  knife ssl check -c 
``

* * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * 

[2014-07-14T17:12:31+00:00] INFO: *** Chef 11.12.8 ***
[2014-07-14T17:12:31+00:00] INFO: Chef-client pid: 3311
[2014-07-14T17:12:33+00:00] WARN: Run List override has been provided.
[2014-07-14T17:12:33+00:00] WARN: Original Run List: []
[2014-07-14T17:12:33+00:00] WARN: Overridden Run List: [recipe[chef-purge]]
[2014-07-14T17:12:33+00:00] INFO: Run List is [recipe[chef-purge]]
[2014-07-14T17:12:33+00:00] INFO: Run List expands to [chef-purge]
[2014-07-14T17:12:33+00:00] INFO: Starting Chef Run for vagrant-ubuntu-precise-64
[2014-07-14T17:12:33+00:00] INFO: Running start handlers
[2014-07-14T17:12:33+00:00] INFO: Start handlers complete.
[2014-07-14T17:12:33+00:00] INFO: HTTP Request Returned 404 Not Found : Object not found: /reports/nodes/vagrant-ubuntu-precise-64/runs
[2014-07-14T17:12:33+00:00] INFO: Loading cookbooks [chef-purge@0.1.0]
[2014-07-14T17:12:33+00:00] INFO: Processing file[/root/x.txt] action create (chef-purge::default line 1)
[2014-07-14T17:12:33+00:00] INFO: Processing package[chef] action purge (chef-purge::default line 6)
[2014-07-14T17:12:35+00:00] INFO: package[chef] purged
[2014-07-14T17:12:35+00:00] WARN: Skipping final node save because override_runlist was given
[2014-07-14T17:12:35+00:00] INFO: Chef Run complete in 1.445303952 seconds
[2014-07-14T17:12:35+00:00] INFO: Running report handlers
[2014-07-14T17:12:35+00:00] INFO: Report handlers complete
---> Chef install status:
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                             Version                           Description
+++-================================-=================================-==============================================================================
un  chef                             <none>                            (no description available)
19:12:40 ∅>
```

## Conclusion

1. It is possible to install the Chef client without an internet connection, but unfortunately Chef is not available via normal package systems like apt and yum. 

2. Chef binaries can be un-installed with a Chef run

The full demo is available at [Github](https://github.com/chris-rock/chef-purge-demo/). Do you have better approaches for running Chef runs without internet?

If you have any questions contact me via [Twitter @chri_hartmann](https://twitter.com/chri_hartmann) or [Github](https://github.com/chris-rock)