---
title: Managing your mainframe infrastructure using Chef
author: chris
date: 2015-01-22
template: article.jade
---

This blog post will focus on running Chef cookbooks on AIX server. As an example we will install various Linux tools via Chef and demonstrate, how a mainframe setup can be automated.

## Introduction

Recently, I published an article about running [Chef on AIX](http://lollyrock.com/articles/ibm-aix-chef/). I worked the last months with Chef and AIX. For my evaluations I used the [IBM Power Development Cloud](http://www.ibm.com/partnerworld/pdp). Using various AIX machines, I needed a setup, where I was able to turn a vanilla AIX system into an perfectly configured system. One part of this setup was the installation of Linux tools to work faster via command line. IBM provides pre-compiled [Linux tools for AIX](http://www-03.ibm.com/systems/power/software/aix/linux/). Since AIX lacks a package manager for rpm packages, the standard method was to run bash scripts that include the dependency graph. In the following sections, I'm going to install basic packages like wget, curl and zip via Chef and manually. 

## The Chef way

As a first step we need to install the Chef client. The best way is via the standard install script provided at `https://www.chef.io/download-chef-client/`. Since the current fix by [Julian Dunn](https://github.com/opscode/opscode-omnitruck/pull/72) for AIX is not yet live, you can use this [script](http://lollyrock.com/articles/managing-mainframe-using-chef/install.sh) instead. I do the following steps to run the script the first time: 

1. Open Terminal
2. Copy content of install.sh into clipboard
3. cat > install.sh, paste content, CTRL-C
4. sh install.sh

After running the script, Chef is installed on your machine. Tip: For production use, you can host the Chef client binaries on your own server and change the script with the required download urls.

The next step is to transfer the cookbooks on your machine. In production environment you could easily copy the cookbooks from a NFS volume for Chef-Solo or use a Chef Server (recommended way).

I wrote a [AIX setup cookbook](https://github.com/chris-rock/aix-base-setup) that offers chef recipes for installing wget, curl, vim, zip and tar. The [recipe for vim](https://github.com/chris-rock/aix-base-setup/blob/master/recipes/vim.rb) looks like:

```ruby
%w(vim-common vim-enhanced).each do |pack|
  aix_toolboxpackage pack do
    action :install
  end
end
```

You find all other recipes in my [Github repo](https://github.com/chris-rock/aix-base-setup/tree/master/recipes). 

On IBM PDC I download the required cookbooks via Chef. Since `aix-base-setup` depends on `aix` cookbook, we need to download multiple cookbooks for setup. I wrote a small Chef recipe to do just that. Create a new file `download.rb` with the following content:

```ruby

%w(https://supermarket.chef.io/cookbooks/aix/download 
   https://supermarket.chef.io/cookbooks/aix-base-setup/download
).each do |cb|
  # download
  remote_file 'download cookbook' do
    source cb
    path '/var/chef/cookbooks/cookbook.tar.gz'
    notifies :run, 'execute[extract-cookbook]', :immediately
  end

  # extract
  execute 'extract-cookbook' do
    cwd '/var/chef/cookbooks/'
    command 'gzip -d < /var/chef/cookbooks/cookbook.tar.gz | tar xvf -'
    action :nothing
  end
end

```

Now run `chef-apply download.rb`. Afterwards the extracted cookbooks are located in `/var/chef/cookbooks/`

```bash
$ > mkdir /var/chef/cookbooks
$ > chef-apply download.rb
[2015-01-29T08:04:38-06:00] WARN: Please install an English UTF-8 locale for Chef to use, falling back to C locale and disabling UTF-8 support.
[2015-01-29T08:04:43-06:00] WARN: Cloning resource attributes for remote_file[download cookbook] from prior resource (CHEF-3694)
[2015-01-29T08:04:43-06:00] WARN: Previous remote_file[download cookbook]: download.rb:5:in `block in run_chef_recipe'
[2015-01-29T08:04:43-06:00] WARN: Current  remote_file[download cookbook]: download.rb:5:in `block in run_chef_recipe'
[2015-01-29T08:04:43-06:00] WARN: Cloning resource attributes for execute[extract-cookbook] from prior resource (CHEF-3694)
[2015-01-29T08:04:43-06:00] WARN: Previous execute[extract-cookbook]: download.rb:12:in `block in run_chef_recipe'
[2015-01-29T08:04:43-06:00] WARN: Current  execute[extract-cookbook]: download.rb:12:in `block in run_chef_recipe'
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * remote_file[download cookbook] action create
    - create new file /var/chef/cookbooks/cookbook.tar.gz
    - update content in file /var/chef/cookbooks/cookbook.tar.gz from none to 2b736b
    (new content is binary, diff output suppressed)
  * execute[extract-cookbook] action run
    - execute gzip -d < /var/chef/cookbooks/cookbook.tar.gz | tar xvf -
  * execute[extract-cookbook] action run
    - execute gzip -d < /var/chef/cookbooks/cookbook.tar.gz | tar xvf -
  * execute[extract-cookbook] action nothing (skipped due to action :nothing)
  * remote_file[download cookbook] action create
    - update content in file /var/chef/cookbooks/cookbook.tar.gz from 2b736b to 5d59e4
    (current file is binary, diff output suppressed)
  * execute[extract-cookbook] action run
    - execute gzip -d < /var/chef/cookbooks/cookbook.tar.gz | tar xvf -
  * execute[extract-cookbook] action run
    - execute gzip -d < /var/chef/cookbooks/cookbook.tar.gz | tar xvf -
  * execute[extract-cookbook] action nothing (skipped due to action :nothing)
$ >
``` 

To apply the `aix-base-setup`, we create a `solo.json` and `solo.rb`. 

```bash
cd /var/chef/
cat > solo.json <<EOF
{
    "run_list":[
        "recipe[aix-base-setup]"
    ]
}
EOF

cat > solo.rb <<EOF
root = File.absolute_path(File.dirname(__FILE__))
node_name "localhost"
file_cache_path root
cookbook_path [ root + '/cookbooks', root + '/site-cookbooks' ]
EOF
```

All required files to run the coobooks are configured on the AIX machine. To install `curl`, `wget`, `vim`, `zip` and `zsh` we start the Chef run. 

```bash
$ > cd /var/chef/
$ > chef-solo -c solo.rb -j solo.json
[2015-01-29T08:07:05-06:00] WARN: Please install an English UTF-8 locale for Chef to use, falling back to C locale and disabling UTF-8 support.
Starting Chef Client, version 12.0.3
Compiling Cookbooks...
Converging 9 resources
Recipe: aix-base-setup::curl
  * aix_toolboxpackage[curl] action install
    * remote_file[/var/chef/curl-7.9.3-2.aix4.3.ppc.rpm] action create
      - create new file /var/chef/curl-7.9.3-2.aix4.3.ppc.rpm
      - update content in file /var/chef/curl-7.9.3-2.aix4.3.ppc.rpm from none to 34b591
      (new content is binary, diff output suppressed)
    * rpm_package[curl] action install
      - install version 7.9.3-2 of package curl
  
Recipe: aix-base-setup::wget
  * aix_toolboxpackage[wget] action install
    * remote_file[/var/chef/wget-1.9.1-1.aix5.1.ppc.rpm] action create
      - create new file /var/chef/wget-1.9.1-1.aix5.1.ppc.rpm
      - update content in file /var/chef/wget-1.9.1-1.aix5.1.ppc.rpm from none to d1274c
      (new content is binary, diff output suppressed)
    * rpm_package[wget] action install
      - install version 1.9.1-1 of package wget
  
Recipe: aix-base-setup::vim
  * aix_toolboxpackage[vim-common] action install
    * remote_file[/var/chef/vim-common-6.3-1.aix5.1.ppc.rpm] action create
      - create new file /var/chef/vim-common-6.3-1.aix5.1.ppc.rpm
      - update content in file /var/chef/vim-common-6.3-1.aix5.1.ppc.rpm from none to 8d0c9f
      (new content is binary, diff output suppressed)
    * rpm_package[vim-common] action install
      - install version 6.3-1 of package vim-common
  
  * aix_toolboxpackage[vim-enhanced] action install
    * remote_file[/var/chef/vim-enhanced-6.3-1.aix5.1.ppc.rpm] action create
      - create new file /var/chef/vim-enhanced-6.3-1.aix5.1.ppc.rpm
      - update content in file /var/chef/vim-enhanced-6.3-1.aix5.1.ppc.rpm from none to 0b6e60
      (new content is binary, diff output suppressed)
    * rpm_package[vim-enhanced] action install
      - install version 6.3-1 of package vim-enhanced
  
Recipe: aix-base-setup::zip
  * aix_toolboxpackage[zip] action install
    * remote_file[/var/chef/zip-2.3-3.aix4.3.ppc.rpm] action create
      - create new file /var/chef/zip-2.3-3.aix4.3.ppc.rpm
      - update content in file /var/chef/zip-2.3-3.aix4.3.ppc.rpm from none to 195196
      (new content is binary, diff output suppressed)
    * rpm_package[zip] action install
      - install version 2.3-3 of package zip
  
  * aix_toolboxpackage[unzip] action install
    * remote_file[/var/chef/unzip-5.51-1.aix5.1.ppc.rpm] action create
      - create new file /var/chef/unzip-5.51-1.aix5.1.ppc.rpm
      - update content in file /var/chef/unzip-5.51-1.aix5.1.ppc.rpm from none to 1c0445
      (new content is binary, diff output suppressed)
    * rpm_package[unzip] action install
      - install version 5.51-1 of package unzip
  
Recipe: aix-base-setup::zsh
  * aix_toolboxpackage[coreutils] action install
    * remote_file[/var/chef/coreutils-5.2.1-2.aix5.1.ppc.rpm] action create
      - create new file /var/chef/coreutils-5.2.1-2.aix5.1.ppc.rpm
      - update content in file /var/chef/coreutils-5.2.1-2.aix5.1.ppc.rpm from none to 1dec06
      (new content is binary, diff output suppressed)
    * rpm_package[coreutils] action install
      - install version 5.2.1-2 of package coreutils
  
  * aix_toolboxpackage[grep] action install
    * remote_file[/var/chef/grep-2.5.1-1.aix4.3.ppc.rpm] action create
      - create new file /var/chef/grep-2.5.1-1.aix4.3.ppc.rpm
      - update content in file /var/chef/grep-2.5.1-1.aix4.3.ppc.rpm from none to 805523
      (new content is binary, diff output suppressed)
    * rpm_package[grep] action install
      - install version 2.5.1-1 of package grep
  
  * aix_toolboxpackage[zsh] action install
    * remote_file[/var/chef/zsh-4.0.4-3.aix5.1.ppc.rpm] action create
      - create new file /var/chef/zsh-4.0.4-3.aix5.1.ppc.rpm
      - update content in file /var/chef/zsh-4.0.4-3.aix5.1.ppc.rpm from none to 3eb16c
      (new content is binary, diff output suppressed)
    * rpm_package[zsh] action install
      - install version 4.0.4-3 of package zsh
  

Running handlers:
Running handlers complete
Chef Client finished, 27/27 resources updated in 35.332408 seconds
$ >
```

We are ready to use the system with a basic setup.

## The classic way

For comparison, I will highlight the installation of the same tools with a manual installation. First, we need to ensure that we are able to install rpm packages. RPM can be downloaded from [IBM's FTP server](ftp://public.dhe.ibm.com/aix/freeSoftware/aixtoolbox/INSTALLP/ppc/rpm.rte).

```bash
l5a1vp051_pub[/home/u0015844] > ftp ftp.software.ibm.com
Connected to dispsd-40-www3.boulder.ibm.com.
220 service.boulder.ibm.com FTP server (Version wu-2.6.2.1(5) Custom Tue Aug 17 13:28:23 MDT 2010) ready.
Name (ftp.software.ibm.com:u0015844): ftp
331 Guest login ok, send any password.
Password:
230 Guest login ok, access restrictions apply.
ftp> cd aix/freeSoftware/aixtoolbox/INSTALLP/ppc
250 CWD command successful.
ftp> binary
200 Type set to I.
ftp> get rpm.rte
200 PORT command successful.
150 Opening BINARY mode data connection for rpm.rte (2408448 bytes).
226 Transfer complete.
2408448 bytes received in 1.299 seconds (1811 Kbytes/s)
local: rpm.rte remote: rpm.rte
ftp> quit
221-You have transferred 2408448 bytes in 1 files.
221-Total traffic for this session was 2410547 bytes in 1 transfers.
221-Thank you for using the FTP service on service.boulder.ibm.com.
221 Goodbye.

installp -qaXgd rpm.rte rpm.rte
+-----------------------------------------------------------------------------+
                    Pre-installation Verification...
+-----------------------------------------------------------------------------+
Verifying selections...done
Verifying requisites...done
Results...

WARNINGS
--------
  Problems described in this section are not likely to be the source of any
  immediate or serious failures, but further actions may be necessary or
  desired.

  Already Installed
  -----------------
  The number of selected filesets that are either already installed
  or effectively installed through superseding filesets is 1.  See
  the summaries at the end of this installation for details.

  NOTE:  Base level filesets may be reinstalled using the "Force"
  option (-F flag), or they may be removed, using the deinstall or
  "Remove Software Products" facility (-u flag), and then reinstalled.

  << End of Warning Section >>

+-----------------------------------------------------------------------------+
                   BUILDDATE Verification ...
+-----------------------------------------------------------------------------+
Verifying build dates...done
FILESET STATISTICS
------------------
    1  Selected to be installed, of which:
        1  Already installed (directly or via superseding filesets)
  ----
    0  Total to be installed


Pre-installation Failure/Warning Summary
----------------------------------------
Name                      Level           Pre-installation Failure/Warning
-------------------------------------------------------------------------------
rpm.rte                   3.0.5.52        Already installed

l5a1vp051_pub[/home/u0015844] >
```bash

Once we have FTP, we are ready to download wget to easy the installation of the following tools. 

```bash
l5a1vp051_pub[/home/u0015844] > ftp ftp.software.ibm.com
Connected to dispsd-40-www3.boulder.ibm.com.
220 service.boulder.ibm.com FTP server (Version wu-2.6.2.1(5) Custom Tue Aug 17 13:28:23 MDT 2010) ready.
Name (ftp.software.ibm.com:u0015844): ftp
331 Guest login ok, send any password.
Password:
230 Guest login ok, access restrictions apply.
ftp> cd aix/freeSoftware/aixtoolbox/RPMS/ppc/wget
250 CWD command successful.
ftp> binary
200 Type set to I.
ftp> get wget-1.9.1-3.aix6.1.ppc.rpm
200 PORT command successful.
150 Opening BINARY mode data connection for wget-1.9.1-3.aix6.1.ppc.rpm (465606 bytes).
226 Transfer complete.
465606 bytes received in 0.8058 seconds (564.3 Kbytes/s)
local: wget-1.9.1-3.aix6.1.ppc.rpm remote: wget-1.9.1-3.aix6.1.ppc.rpm
ftp> quit
221-You have transferred 465606 bytes in 1 files.
221-Total traffic for this session was 468128 bytes in 2 transfers.
221-Thank you for using the FTP service on service.boulder.ibm.com.
221 Goodbye.
l5a1vp051_pub[/home/u0015844] > rpm -hUv wget-1.9.1-3.aix6.1.ppc.rpm
wget                        ##################################################
l5a1vp051_pub[/home/u0015844] >
```

Since wget is available on AIX now, we can use it to download all required rpm packages.

```bash
# Download the vim binaries
wget -r -nd ftp://ftp.software.ibm.com/aix/freeSoftware/aixtoolbox/RPMS/ppc/vim/vim-common-6.3-1.aix5.1.ppc.rpm
wget -r -nd ftp://ftp.software.ibm.com/aix/freeSoftware/aixtoolbox/RPMS/ppc/vim/vim-enhanced-6.3-1.aix5.1.ppc.rpm
# Install vim
rpm -hUv vim-common-6.3-1.aix5.1.ppc.rpm vim-enhanced-6.3-1.aix5.1.ppc.rpm
vim-common                  ##################################################
vim-enhanced                ##################################################
```

```bash
# Download & Install curl
wget -r -nd ftp://ftp.software.ibm.com/aix/freeSoftware/aixtoolbox/RPMS/ppc/curl/curl-7.9.3-2.aix4.3.ppc.rpm
# Install vim
rpm -hUv curl-7.9.3-2.aix4.3.ppc.rpm
```

```bash
# Download & Install the zsh binaries
wget -r -nd ftp://ftp.software.ibm.com/aix/freeSoftware/aixtoolbox/RPMS/ppc/zsh/zsh-4.0.4-3.aix5.1.ppc.rpm
wget -r -nd ftp://ftp.software.ibm.com/aix/freeSoftware/aixtoolbox/RPMS/ppc/coreutils/coreutils-5.2.1-2.aix5.1.ppc.rpm
wget -r -nd ftp://ftp.software.ibm.com/aix/freeSoftware/aixtoolbox/RPMS/ppc/grep/grep-2.5.1-1.aix4.3.ppc.rpm
rpm -hUv zsh-4.0.4-3.aix5.1.ppc.rpm coreutils-5.2.1-2.aix5.1.ppc.rpm grep-2.5.1-1.aix4.3.ppc.rpm
```

```bash
# Download & Install zip
wget -r -nd ftp://ftp.software.ibm.com/aix/freeSoftware/aixtoolbox/RPMS/ppc/zip/zip-2.3-3.aix4.3.ppc.rpm
wget -r -nd ftp://ftp.software.ibm.com/aix/freeSoftware/aixtoolbox/RPMS/ppc/unzip/unzip-5.51-1.aix5.1.ppc.rpm
su root -c rpm -hUv zip-2.3-3.aix4.3.ppc.rpm unzip-5.51-1.aix5.1.ppc.rpm
```

Finally, we have reached the final setup.

## Conclusion

Both, the manual and Chef approach, are valid ways to setup a machine. If you need to setup 5 packages, the effort is nearly the same. Chef will become more efficient if you go further and start changing system and application configuration as well as installing more packages. It will become very handy to manage the setup in Chef, because the manual steps required on the system keep the same, no matter how many steps are involved in your cookbooks. This improves the deployment time and decreases the time of reoccurring tasks.

Questions:

- Why not use knife bootstrap?

Currently, knife bootstrap does not work with AIX, I opened a [ticket](https://github.com/chef/chef/issues/2836) to address this issue. Additionally I had to work on systems that are only reachable via jump hosts. The Chef-Solo approach helped to use the power of Chef within existing infrastructure. Ops teams can use the existing infrastructure and migrate slowly to an environment where the complete setup is managed via Configuration Management tools.

If you have any questions contact me via [Twitter @chri_hartmann](https://twitter.com/chri_hartmann) or [Github](https://github.com/chris-rock)

## References

- [AIX Toolbox for Linux Applications](http://www-03.ibm.com/systems/power/software/aix/linux/toolbox/altlic.html)
- [IBM AIX Toolbox download informationn](http://www-03.ibm.com/systems/power/software/aix/linux/toolbox/download.html)
- [AIX Toolbox for Linux Applications README](ftp://public.dhe.ibm.com/aix/freeSoftware/aixtoolbox/README.txt)
- [AIX Toolbox Repositoryy](ftp://ftp.software.ibm.com/aix/freeSoftware/aixtoolbox/RPMS/ppc/)
- [AIX for System Administrators](http://aix4admins.blogspot.de/2011/06/commands-oslevel-shows-actual-bos-level.html)
