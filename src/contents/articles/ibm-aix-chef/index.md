---
title: Chef loves AIX - Mainframe Automation
author: chris
date: 2014-12-03
template: article.jade
---

I was very exited to play with IBM AIX and Chef 12. Coming from an Open Stack background with deep knowledge about automation and security with RedHat Linux and Windows Server, I was looking forward to gain insights about using the configuration management tool Chef with AIX. I have done some big deployments on cloud infrastructures and I am very pleased to see some technologies finding their way into core enterprise applications.

## Introduction

With Chef, you can automate how you build, deploy, and manage your infrastructure. Your infrastructure becomes versionable, testable, and repeatable as application code which is especially relevant if you maintain a mission-critical system. Every deployment is required to go through a rigid approval process via staging environments to ensure that the customer will not notice any service interruptions.

AIX is a great platform for such mission critical systems, because it provides a high level of performance and reliability. Due to the fact, that it runs on [IBM Power Systems](http://www.ibm.com/systems/power/), it is also very scalable.

Before I walk you through the details of Chef on AIX, I would like to thank [Julian from Chef Inc.](https://twitter.com/julian_dunn) for his exceptional support over the last months. Additionally, I like to thank IBM for providing the amazing [IBM Power Development Cloud](www.ibm.com/partnerworld/pdp).

## Getting started with Chef on AIX

The following tests are executed on vanilla AIX 7.1. I installed some GNU tools like vim, zsh and wget to get a nice console feeling. Finally, I was ready to download and install Chef 12. The AIX version of Chef is supported for AIX 6.1 and AIX 7.1 and you can read more about the announcement at [Chef Lists](http://lists.opscode.com/sympa/arc/chef/2014-11/msg00038.html).

```bash
l5a1vp051_pub > wget -r -nd http://opscode-omnibus-packages.s3.amazonaws.com/aix/6.1/powerpc/chef-12.0.0-rc.0-1.powerpc.bff
--10:18:05--  http://opscode-omnibus-packages.s3.amazonaws.com/aix/6.1/powerpc/chef-12.0.0-rc.0-1.powerpc.bff
           => `chef-12.0.0-rc.0-1.powerpc.bff'
Resolving opscode-omnibus-packages.s3.amazonaws.com... 54.231.64.169
Connecting to opscode-omnibus-packages.s3.amazonaws.com[54.231.64.169]:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 86,565,376 [binary/octet-stream]

100%[=========================================================================================>] 86,565,376    23.55M/s    ETA 00:00

10:18:09 (22.67 MB/s) - `chef-12.0.0-rc.0-1.powerpc.bff' saved [86565376/86565376]


FINISHED --10:18:09--
Downloaded: 86,565,376 bytes in 1 files
l5a1vp051_pub >
```

Chef delivers in a AIX backup file format (similar to unix tar format). To install the content, we need to know the fileset name. We display the content with `installp -ld chef-12.0.0-rc.0-1.powerpc.bff`

```bash
l5a1vp051_pub > installp -ld chef-12.0.0-rc.0-1.powerpc.bff
  Fileset Name                Level                     I/U Q Content
  ====================================================================
  chef                        12.0.0.1                   I  N usr,root
#   The full stack of chef
l5a1vp051_pub > 
```

Now, we collected all required information to start the installation. We use `installp` with the preview flag, to verify that everything works as intended: `installp -apXY -d chef-12.0.0-rc.0-1.powerpc.bff chef`. In case you are not familliar with the flags: `a: apply`, `p: preview`, `X: extend fs` and ` Y: accepts license agreements`.

```bash
l5a1vp051_pub > installp -apXY -d chef-12.0.0-rc.0-1.powerpc.bff chef
*******************************************************************************
installp PREVIEW:  installation will not actually occur.
*******************************************************************************

+-----------------------------------------------------------------------------+
                    Pre-installation Verification...
+-----------------------------------------------------------------------------+
Verifying selections...done
Verifying requisites...done
Results...

SUCCESSES
---------
  Filesets listed in this section passed pre-installation verification
  and will be installed.

  Selected Filesets
  -----------------
  chef 12.0.0.1                               # The full stack of chef

  << End of Success Section >>

+-----------------------------------------------------------------------------+
                   BUILDDATE Verification ...
+-----------------------------------------------------------------------------+
Verifying build dates...done
FILESET STATISTICS
------------------
    1  Selected to be installed, of which:
        1  Passed pre-installation verification
  ----
    1  Total to be installed

RESOURCES
---------
  Estimated system resource requirements for filesets being installed:
                (All sizes are in 512-byte blocks)
      Filesystem                     Needed Space             Free Space
      /                                   15676                 363384
      /usr                                37276                 380312
      /opt                               291496                 132640
      -----                            --------                 ------
      TOTAL:                             344448                 876336

  NOTE:  "Needed Space" values are calculated from data available prior
  to installation.  These are the estimated resources required for the
  entire operation.  Further resource checks will be made during
  installation to verify that these initial estimates are sufficient.

******************************************************************************
End of installp PREVIEW.  No apply operation has actually occurred.
******************************************************************************
l5a1vp051_pub >
```

Since we are sure now, that the installation works, we remove the preview flag and finalize the installation.

```
l5a1vp051_pub > installp -aXY -d chef-12.0.0-rc.0-1.powerpc.bff chef
+-----------------------------------------------------------------------------+
                    Pre-installation Verification...
+-----------------------------------------------------------------------------+
Verifying selections...done
Verifying requisites...done
Results...

SUCCESSES
---------
  Filesets listed in this section passed pre-installation verification
  and will be installed.

  Selected Filesets
  -----------------
  chef 12.0.0.1                               # The full stack of chef

  << End of Success Section >>

+-----------------------------------------------------------------------------+
                   BUILDDATE Verification ...
+-----------------------------------------------------------------------------+
Verifying build dates...done
FILESET STATISTICS
------------------
    1  Selected to be installed, of which:
        1  Passed pre-installation verification
  ----
    1  Total to be installed

Filesystem size changed to 2097152
+-----------------------------------------------------------------------------+
                         Installing Software...
+-----------------------------------------------------------------------------+

installp:  APPLYING software for:
        chef 12.0.0.1

Restoring files, please wait.
14485 files restored.
Thank you for installing Chef!
Finished processing all filesets.  (Total time:  2 mins 4 secs).

+-----------------------------------------------------------------------------+
                                Summaries:
+-----------------------------------------------------------------------------+

Installation Summary
--------------------
Name                        Level           Part        Event       Result
-------------------------------------------------------------------------------
chef                        12.0.0.1        USR         APPLY       SUCCESS
chef                        12.0.0.1        ROOT        APPLY       SUCCESS

l5a1vp051_pub > chef-client -v
Chef: 12.0.0.rc.0
```

The chef-client is successfully installed now.

## Hello World

Now we want to ensure that the system works. Therefore we create a simple hello world cookbook that creates a file `x.txt` in the home directory and prints `HELLO WORLD AIX.`

```bash
$ mkdir -p  cookbooks/helloworld/recipes/
$ cd cookbooks
$ cat > helloworld/recipes/default.rb
file "#{ENV['HOME']}/x.txt" do
  content 'HELLO WORLD AIX'
end
```

We use chef-zero to run the helloworld cookbook.

```bash
$ chef-client -z -o helloworld
[2014-12-03T08:14:41-06:00] WARN: No config file found or specified on command line, using command line options.
Starting Chef Client, version 12.0.0.rc.0
[2014-12-03T08:14:46-06:00] WARN: Run List override has been provided.
[2014-12-03T08:14:46-06:00] WARN: Original Run List: []
[2014-12-03T08:14:46-06:00] WARN: Overridden Run List: [recipe[helloworld]]
resolving cookbooks for run list: ["helloworld"]
Synchronizing Cookbooks:
  - helloworld
Compiling Cookbooks...
Converging 1 resources
Recipe: helloworld::default
  * file[/home/u0015844/x.txt] action create
    - create new file /home/u0015844/x.txt
    - update content in file /home/u0015844/x.txt from none to 787ec7
    --- /home/u0015844/x.txt        2014-12-03 08:14:46.000000000 -0600
    +++ /home/u0015844/.x.txt20141203-10027096-1qnz8tq      2014-12-03 08:14:46.000000000 -0600
    @@ -1 +1,2 @@
    +HELLO WORLD AIX
[2014-12-03T08:14:46-06:00] WARN: Skipping final node save because override_runlist was given

Running handlers:
Running handlers complete
Chef Client finished, 1/1 resources updated in 3.754716 seconds
l5a1vp051_pub >
```

<p>
<center><img src="/articles/ibm-aix-chef/hellochef.png" alt="A Chef run on AIX 7.1" title="Chef on AIX" width="700px"></center></p>

Since we know that Chef works very well on AIX, let's start deploying more applications.

Follow my profile on [Twitter](https://twitter.com/chri_hartmann) or [Github](https://github.com/chris-rock/followers)
