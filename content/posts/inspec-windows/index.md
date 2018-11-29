---
title: Windows Infrastructure Testing and Compliance with InSpec
author: Christoph Hartmann
date: 2017-04-10
tags:
  - inspec
aliases:
  - /articles/inspec-windows/
---

InSpec is an infrastructure testing and compliance tool that allows you to write re-usable tests for your IT components. InSpec tests can easily be used in development and production environments to shift [Compliance left](https://blog.chef.io/2016/09/26/shift-left-security-and-compliance-automation-with-inspec-and-chef/). This blog post will highlight how you can leverage InSpec on Windows.

## Install InSpec on Windows

First things first: We need InSpec on our workstation. There are two packages that offer an easy way to get started. For production and standalone environments, I recommend the InSpec package. Alternatively there is ChefDK, if you need Chef + Test-Kitchen + InSpec. You can download both packages from [https://downloads.chef.io/](https://downloads.chef.io/).

Another option is to install InSpec via a Powershell script:

```powershell
. { iwr -useb https://omnitruck.chef.io/install.ps1 } | iex; install -project inspec
```

Once InSpec is installed, run `inspec version` to verify that the installation was successful.

## Write your first test

In my case, I am going to use [Visual Studio Code](https://code.visualstudio.com/) to write the InSpec test. Open Powershell and create a new directory:

```ruby
# create a directory for the test
md inspec
cd .\inspec
# open Visual Studio Code
code .
```

{{< figure src="inspec-create-dir.png" alt="Create new directory for InSpec tests" title="Create new directory for InSpec tests" >}}

Now, create a new file in Visual Studio Code and write down the first InSpec test:

```ruby
describe service('DHCP Client') do
  it { should be_installed }
  it { should be_running }
end
```

This is all we need to test if the `DHCP Client` service is installed and running. Save the file and let's execute the test.

{{< figure src="vscode-test.png" alt="Write InSpec tests in VSCode" title="Write InSpec tests in VSCode" >}}

Execute InSpec from Powershell:

```bash
inspec exec .\test.rb
```

{{< figure src="inspec-exec-test.png" alt="Execute InSpec test" title="Execute InSpec test" >}}

## Run InSpec tests against a remote Windows machine

I passed the `hello world` of InSpec by running the first test on our workstation. As a next step, we run the same test against a remote Windows 2012 R2 server. We will use the same InSpec command with additional target information:

```bash
inspec exec test.rb -t winrm://Administarator@hostname -p 'P@ssword'
```

{{< figure src="inspec-exec-remote.png" alt="Execute InSpec test remotely" title="Execute InSpec test remotely" >}}

With a few commands, we executed an InSpec test against a local workstation and a remote server.

## Resources for Windows

Above, we explained how an InSpec test is created and executed. To make this experience great, InSpec ships with a number of resources optimized for Windows environments. Those resources range from core operating system essentials to application components.

Verify Windows settings and configuration:

* [file](http://inspec.io/docs/reference/resources/file/)
* [package](http://inspec.io/docs/reference/resources/package/)
* [port](http://inspec.io/docs/reference/resources/port/)
* [registry_key](http://inspec.io/docs/reference/resources/registry_key/)
* [service](http://inspec.io/docs/reference/resources/service/)
* [user](http://inspec.io/docs/reference/resources/user/)
* [users](http://inspec.io/docs/reference/resources/users/)
* [windows_feature](http://inspec.io/docs/reference/resources/windows_feature/)
* [windows_task](http://inspec.io/docs/reference/resources/windows_task/)
* [wmi](http://inspec.io/docs/reference/resources/wmi/)

Audit Windows security settings:

* [audit_policy](http://inspec.io/docs/reference/resources/audit_policy/)
* [security_policy](http://inspec.io/docs/reference/resources/security_policy/)

Run custom scripts:

* [powershell](http://inspec.io/docs/reference/resources/powershell/)
* [vbscript](http://inspec.io/docs/reference/resources/vbscript/)

Verify IIS configuration:

* [iis_site](http://inspec.io/docs/reference/resources/iis_site/)

More resources are available and documented at [InSpec Docs](http://inspec.io/docs/reference/resources/).

## Verify configuration of the Windows operating system

The following example illustrate the use of the `file`, `user`, `users`, `package` and `windows_taks` resource.

<script src="https://gist.github.com/chris-rock/3ab57d7d1bb3d1b813f614f81dcfafbf.js"></script>

The result on my Windows 10 workstation will return:

{{< figure src="test-win.png" alt="Run InSpec operating system checks" title="Run InSpec operating system checks" >}}

## Security checks for Windows

If you are interested in operating system hardening for Windows, you need to be able to verify `registry_key`, `security_policy` or `audit_policy`.

<script src="https://gist.github.com/chris-rock/7269ebfbff4f2500e59f922aa9d598fa.js"></script>

Have a look at [Administer Security Policy Settings](https://technet.microsoft.com/en-us/library/jj966254.aspx),[User Rights Assignment](https://technet.microsoft.com/en-us/library/dn221963.aspx), [Privilege Constants](https://msdn.microsoft.com/en-us/library/windows/desktop/bb530716.aspx) and [Audit Policy](https://technet.microsoft.com/en-us/library/cc766468.aspx) to learn more about possible settings.

{{< figure src="test-sec.png" alt="Run security checks" title="Run security checks" >}}

## Reuse existing Powershell or VBScript in InSpec

InSpec also supports running Powershell and VBScript. This allows you to re-use existing scripts inside InSpec. `powershell` and `vbscript` resources provide the required capabilities to embed scripts in InSpec.

```ruby
script = <<-EOH
  Write-Output 'hello'
EOH

describe powershell(script) do
  its('stdout') { should eq "hello\r\n" }
  its('stderr') { should eq '' }
end
```

Besides Powershell, InSpec supports `vbscript`, too:

```ruby
script = <<-EOH
  WScript.Echo "hello"
EOH

describe vbscript(script) do
  its('stdout') { should eq "hello\r\n" }
end
```

## Application testing with InSpec

In addition to operating system checks, we can test IIS configurations with InSpec as well. The following example demonstrates tests for an out-of-the-box IIS server.

<script src="https://gist.github.com/chris-rock/a079adf369c88ac671b2b0f96f9bb229.js"></script>

## Compliance

Above, we introduced InSpec as an infrastructure testing tool. For compliance, [additional metadata](http://inspec.io/docs/reference/dsl_inspec/) can be attached around `describe` tests. The following example shows the use of criticality (`impact`), `title`, tags (`tag`) and references (`ref`).

```ruby
control 'windows-base-102' do
  impact 1.0
  title 'Anonymous Access to Windows Shares and Named Pipes is Disallowed'
  tag cis: ['windows_2012r2:2.3.11.8', 'windows_2016:2.3.10.9']
  ref 'CIS Microsoft Windows Server 2012 R2 Benchmark'
  ref 'CIS Microsoft Windows Server 2016 RTM (Release 1607) Benchmark'
  describe registry_key('HKLM\System\CurrentControlSet\Services\LanManServer\Parameters') do
    it { should exist }
    its('RestrictNullSessAccess') { should eq 1 }
  end
end
```

In most cases, you do not want to start from scratch to develop compliance benchmarks. The [DevSec.io](http://dev-sec.io/) project already provides industry best-practices for Linux and Windows operating systems. The Windows benchmark is currently in development and contributions are welcome to cover more areas. Let's run the DevSec Windows Baseline quickly:

```ruby
inspec exec https://github.com/dev-sec/windows-baseline
```

Are you compliant?

{{< figure src="test-devsec.png" alt="Run DevSec Windows Baseline" title="Run DevSec Windows Baseline" >}}

I recommend you have a look at the following DevSec profiles:

 - [DevSec Windows Baseline](https://github.com/dev-sec/windows-baseline)
 - [DevSec Windows Patch Baseline](https://github.com/dev-sec/windows-patch-baseline)

Those benchmark already support Windows 2012+, Windows 2016 and Windows Nano.

## Summary

This blog post covered the following:

 * InSpec installation on Windows
 * InSpec resources for Windows
 * Execute InSpec tests locally and remotely
 * InSpec Compliance profiles for Windows

You should be prepared to start your first tests with InSpec in your own environment. If you feel that something is missing, please let me know.
