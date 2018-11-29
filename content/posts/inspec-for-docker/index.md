---
title: InSpec for Docker
author: Christoph Hartmann
date: 2017-05-01
tags:
  - inspec
  - docker
aliases:
  - /articles/inspec-for-docker/
---

Docker environments enable you to manage fast-moving infrastructure. The faster you move, the better your test environment needs to be. InSpec provides that capability. With the recent addition of 3 new resources: `docker`, `docker_image` and a `docker_container`, it became even easier to verify docker hosts and docker containers. This blog post demonstrates how to use InSpec to verify your Docker environments.

## Overview

{{< figure src="inspec-docker.png" alt="Docker Architecture Overview" title="Docker Architecture Overview" >}}

Before I am talking about docker infrastructure testing, I need to explain the different parts of the docker universe quickly.

{{< figure src="docker-architecture.svg" alt="Docker Architecture Overview" title="Docker Architecture Overview Source: https://docs.docker.com/engine/docker-overview/#docker-architecture" >}}


A docker host is the runtime environment (docker cli + docker daemon) that manages docker images and containers on a machine. Therefore it helps to manages the complete lifecycle of a container and provides a nice CLI to interact. The docker daemon can start multiple containers in parallel. A good read to understand containers is [Containers vs. Zones vs. Jails vs. VMs](https://news.ycombinator.com/item?id=13982620).

InSpec allows you to scan both, the host and the running container. At first, I am demonstrating the use of InSpec with a running container to ensure the container is configured correctly. Afterwards, I am going to talk about 3 InSpec resources that help you to verify a docker host environment:

 - `docker container`
 - `docker image`
 - `docker`

Those resources are heavily used in [DevSec's CIS Docker Benchmark ](https://github.com/dev-sec/cis-docker-benchmark) to ensure best-practices security configuration for docker hosts.

## Verify a running container

The simplest use-case is to verify a running docker container. Just start a new container by

```bash
docker run -it alpine /bin/sh
```

Now, we can use InSpec to run the [DevSec Linux Baseline](https://github.com/dev-sec/linux-baseline) against that container:

```bash
$ inspec exec https://github.com/dev-sec/linux-baseline -t docker://ca2dbee25ddc
Profile: DevSec Linux Security Baseline (linux-baseline)
Version: 2.0.1
Target:  docker://ca2dbee25ddca24f453005a15cd64f1004fbe6f974df957658fcdf6d1fa6331e

  ✔  os-01: Trusted hosts login
     ✔  Command find / -name '.rhosts' stdout should be empty
     ✔  Command find / -name 'hosts.equiv'  stdout should be empty
  ×  os-02: Check owner and permissions for /etc/shadow (1 failed)
     ✔  File /etc/shadow should exist
     ✔  File /etc/shadow should be file
     ✔  File /etc/shadow should be owned by "root"
     ✔  File /etc/shadow should not be executable
     ✔  File /etc/shadow should be writable by owner
     ✔  File /etc/shadow should be readable by owner
     ✔  File /etc/shadow should not be readable by other
     ×  File /etc/shadow group should eq nil

     expected: nil
          got: "shadow"

     (compared using ==)

...

  ✔  sysctl-31a: Secure Core Dumps - dump settings
    ✔  Kernel Parameter fs.suid_dumpable value should cmp == /(0|2)/
  ↺  sysctl-31b: Secure Core Dumps - dump path
    ↺  Skipped control due to only_if condition.
  ✔  sysctl-32: kernel.randomize_va_space
    ✔  Kernel Parameter kernel.randomize_va_space value should eq 2
  ✔  sysctl-33: CPU No execution Flag or Kernel ExecShield
    ✔  /proc/cpuinfo Flags should include NX

  Profile Summary: 20 successful, 23 failures, 7 skipped
  Test Summary: 46 successful, 56 failures, 9 skipped     
```

You can write your own tests and run those against a running container, too. Further information about the available InSpec resources is available at [InSpec Docs](https://www.inspec.io/docs/)

## How to use 'docker_container'

The `docker_container` resource helps you to verify running container. For the following test, I am going to start a new postgres database:

```bash
docker run --name some-postgres -e POSTGRES_PASSWORD=mysecretpassword -d  -p 5432:5432 postgres
```

I'd like to write a test that ensures the container `some-postgres` runs and the port is mapped. A simple InSpec test can do the trick:

```ruby
describe docker_container('some-postgres') do
  it { should exist }
  it { should be_running }
  its('repo') { should eq 'postgres' }
  its('ports') { should eq "0.0.0.0:5432->5432/tcp" }
  its('command') { should eq 'docker-entrypoint.sh postgres' }
end
```

I am going to run that test that quickly in InSpec Shell, but you can use the same test in a `test.rb` and run `inspec exec test.rb`.

```bash
inspec shell
Welcome to the interactive InSpec Shell
To find out how to use it, type: help

You are currently running on:

    OS platform:  mac_os_x
    OS family:  darwin
    OS release: 10.12.4

inspec> describe docker_container('some-postgres') do
inspec>   it { should exist }  
inspec>   it { should be_running }  
inspec>   its('repo') { should eq 'postgres' }  
inspec>   its('ports') { should eq "0.0.0.0:5432->5432/tcp" }  
inspec>   its('command') { should eq 'docker-entrypoint.sh postgres' }  
inspec> end  

Profile: inspec-shell
Version: (not specified)

  Docker Container
     ✔  some-postgres should exist
     ✔  some-postgres should be running
     ✔  some-postgres repo should eq "postgres"
     ✔  some-postgres ports should eq "0.0.0.0:5432->5432/tcp"
     ✔  some-postgres command should eq "docker-entrypoint.sh postgres"

Test Summary: 5 successful, 0 failures, 0 skipped

```

In case you want to verify a specific container id, you can do that as well:

```bash
describe docker_container(id: 'a3df5ff6740f') do
  it { should exist }
  it { should be_running }
end
```

## How to use 'docker_image'

Another use-case I am seeing very often is to ensure a specific Docker image is not used in your environment anymore. I am going to use Ubuntu 12.04 as an example, since it went [end of life](http://fridge.ubuntu.com/2017/03/15/ubuntu-12-04-precise-pangolin-reaches-end-of-life-on-april-28-2017/) recently. The following test ensures that `u12:latest` image is available on your docker host:

```bash
describe docker_image('u12:latest') do
  it { should_not exist }
end
```

Lets run it in InSpec Shell quickly:

```bash
inspec shell
Welcome to the interactive InSpec Shell
To find out how to use it, type: help

You are currently running on:

    OS platform:  mac_os_x
    OS family:  darwin
    OS release: 10.12.4

inspec> describe docker_image('u12:latest') do
inspec>   it { should_not exist }  
inspec> end  

Profile: inspec-shell
Version: (not specified)

  Docker Image
     ✔  u12:latest should not exist

Test Summary: 1 successful, 0 failures, 0 skipped
inspec>
```

The `docker_image` resource can also be used to verify a specific image id:

```bash
describe docker_image('alpine:latest') do
  it { should exist }
  its('id') { should eq 'sha256:4a415e3663882fbc554ee830889c68a33b3585503892cc718a4698e91ef2a526' }
  its('image') { should eq 'alpine:latest' }
  its('repo') { should eq 'alpine' }
  its('tag') { should eq 'latest' }
end
```

In addition you could also verify that no container is using the `u12:latest` image. We ask InSpec to return all images of the containers. This will return a list of images used by the containers. As a next step, we just have to ensure the specific image is not used anymore.

```bash
describe docker.containers do
  its('images') { should_not include 'u12:latest' }
end
```

## More advanced used cases with 'docker' resource

The `docker` resource is more complex to use than `docker_image` and `docker_container`, but allows you to implement more complex query scenarios. It provides 4 key methods to fetch more information about the docker host:

- `containers` returns all running and exited containers by running `docker ps -a --no-trunc` under the hood
- `images` returns all locally available images by running `docker images -a --no-trunc`
- `version` is the parsed content of `docker version`
- `object(id)` returns information about a docker object and parses the output from `docker inspect id`

**Docker Containers**

```bash
describe docker.containers do
  its('images') { should_not include 'u12:latest' }
end
```

Let us inspec the `docker` resource with InSpec Shell. This will show two containers, one is already exited and another one is still running:

```bash
$ inspec shell
dockeWelcome to the interactive InSpec Shell
To find out how to use it, type: help

You are currently running on:

    OS platform:  mac_os_x
    OS family:  darwin
    OS release: 10.12.4

inspec> docker.containers.entries
=> [docker.containers.entries
=> [#<struct
  command="\"/bin/sh\"",
  id="ca2dbee25ddca24f453005a15cd64f1004fbe6f974df957658fcdf6d1fa6331e",
  image="alpine",
  labels="",
  localvolumes="0",
  mounts="",
  names="keen_dijkstra",
  networks="bridge",
  ports="",
  runningfor="About an hour ago",
  size="0B",
  status="Exited (127) 13 seconds ago",
  exists?=nil,
  running?=nil>,
 #<struct
  command="\"docker-entrypoint.sh postgres\"",
  id="921f6363ceaa3c5253a31068a0c8cb7b3a78c1353cf6b4125c7a5460d4e8ce2a",
  image="postgres",
  labels="",
  localvolumes="1",
  mounts="afee92bcd2e8e84cccf884cd42443f0ff718789dca271e7497a308e22622514a",
  names="some-postgres",
  networks="bridge",
  ports="0.0.0.0:5432->5432/tcp",
  runningfor="3 hours ago",
  size="0B",
  status="Up 3 hours",
  exists?=nil,
  running?=nil>]
```

The resource allows you to ask for all running containers:

```bash
docker.containers.running?.entries
=> [#<struct
  command="\"docker-entrypoint.sh postgres\"",
  id="921f6363ceaa3c5253a31068a0c8cb7b3a78c1353cf6b4125c7a5460d4e8ce2a",
  image="postgres",
  labels="",
  localvolumes="1",
  mounts="afee92bcd2e8e84cccf884cd42443f0ff718789dca271e7497a308e22622514a",
  names="some-postgres",
  networks="bridge",
  ports="0.0.0.0:5432->5432/tcp",
  runningfor="3 hours ago",
  size="0B",
  status="Up 3 hours",
  exists?=nil,
  running?=nil>]
```

You can also filter containers by the following fields: `commands`, `ids`, `images`, `labels`, `local_volumes`, `mounts`, `names`, `networks`, `ports`, `running_for`, `sizes`,`status`, `running?`,`exists?`. For example, if you want to verify the configuration for each container, you can do the following in InSpec:

```bash
# returns all container ids
docker.containers.ids.each do |id|
  # call docker inspect to retrieve detailed information about the container
  describe docker.object(id) do
    its('HostConfig.Privileged') { should cmp false }
  end
end
```

If you want to do the following for running containers, just say so:

```ruby
# returns all running container ids
docker.containers.running?.ids.each do |id|
  # call docker inspect to retrieve detailed information about the container
  describe docker.object(id) do
    its('HostConfig.Privileged') { should cmp false }
  end
end
```

If you a looking for a specific container:

```ruby
# returns all running container ids
docker.containers.where { names == 'some-postgres' }.ids.each do |id|
  # call docker inspect to retrieve detailed information about the container
  describe docker.object(id) do
    its('HostConfig.Privileged') { should cmp false }
  end
end
```

**Docker Images**

Docker images can be inspected with a similar approach:

```ruby
describe docker.images do
  its('repositories') { should_not include 'inssecure_image' }
end
```

You can also filter images by the following fields: `ids`, `repositories`, `tags`, `sizes`, `digests`, `created`, `created_since` and `exists?`.


If you want to verify that no image is using the ADD instruction in Dockerfile, you can iterate over all image ids and ask `docker history` if the ADD instruction was used:

```ruby
docker.images.ids.each do |id|
  describe command("docker history #{id}| grep 'ADD'") do
    its('stdout') { should eq '' }
  end
end
```

**Docker Version**

InSpec allows you to verify the docker client and server version that is used on your machine. Behind the scenes, it is using the `docker version` command. This would return the following values:

```bash
$ docker version
Client:
 Version:      17.05.0-ce-rc1
 API version:  1.29
 Go version:   go1.7.5
 Git commit:   2878a85
 Built:        Tue Apr 11 20:55:05 2017
 OS/Arch:      darwin/amd64

Server:
 Version:      17.05.0-ce-rc1
 API version:  1.29 (minimum version 1.12)
 Go version:   go1.7.5
 Git commit:   2878a85
 Built:        Tue Apr 11 20:55:05 2017
 OS/Arch:      linux/amd64
 Experimental: true
```

In InSpec Shell, you see the parsed values:

```bash
inspec> docker.version
=> {"Client"=>
  {"Version"=>"17.05.0-ce-rc1",
   "ApiVersion"=>"1.29",
   "DefaultAPIVersion"=>"1.29",
   "GitCommit"=>"2878a85",
   "GoVersion"=>"go1.7.5",
   "Os"=>"darwin",
   "Arch"=>"amd64",
   "BuildTime"=>"Tue Apr 11 20:55:05 2017"},
 "Server"=>
  {"Version"=>"17.05.0-ce-rc1",
   "ApiVersion"=>"1.29",
   "MinAPIVersion"=>"1.12",
   "GitCommit"=>"2878a85",
   "GoVersion"=>"go1.7.5",
   "Os"=>"linux",
   "Arch"=>"amd64",
   "KernelVersion"=>"4.9.21-moby",
   "Experimental"=>true,
   "BuildTime"=>"Tue Apr 11 20:55:05 2017"}}
```

Now, I am going to write a check that ensures docker is newer than `1.12`

```ruby
describe docker.version do
  its('Server.Version') { should cmp >= '1.12'}
  its('Client.Version') { should cmp >= '1.12'}
end
```

**Docker Info**

`docker.info` parses the output of `docker info`:

```bash
docker info
Containers: 5
 Running: 1
 Paused: 0
 Stopped: 4
Images: 108
Server Version: 17.05.0-ce-rc1
....
```

Again, you can inspect all values in InSpec with InSpec Shell:

```bash
inspec> docker.info
=> {"ID"=>"YMXQ:5C62:Q3AR:XV77:7LS6:O56G:A6VZ:V5LJ:WNNJ:UQTO:QKPB:K62P",
 "Containers"=>5,
 "ContainersRunning"=>1,
 "ContainersPaused"=>0,
 "ContainersStopped"=>4,
 "Images"=>108,
 "Driver"=>"overlay2",
 "DriverStatus"=>[["Backing Filesystem", "extfs"], ["Supports d_type", "true"], ["Native Overlay Diff", "true"]],
 "SystemStatus"=>nil,
 "Plugins"=>{"Volume"=>["local"], "Network"=>["bridge", "host", "ipvlan", "macvlan", "null", "overlay"], "Authorization"=>[]},
 "MemoryLimit"=>true,
 "SwapLimit"=>true,
 "KernelMemory"=>true,
 "CpuCfsPeriod"=>true,
 "CpuCfsQuota"=>true,
 "CPUShares"=>true,
 "CPUSet"=>true,
 "IPv4Forwarding"=>true,
 "BridgeNfIptables"=>true,
 "BridgeNfIp6tables"=>true,
 "Debug"=>true,
 "NFd"=>25,
 "OomKillDisable"=>true,
```

Now, lets ensure we are not running more than 20 containers on a node:

```ruby
describe docker.info do
  its('ContainersRunning') { should cmp <= 20 }
end
```

If Swarm is active, you could verify that you have only a limited amount of leaders:

```ruby
describe docker.info do
  its('Swarm.Managers') { should cmp <= 3 }
end
```

**Docker Object**

Each docker object store multiple attributes. Docker allows you to see those values by running `docker inspect`. This data is parsed in InSpec too and can be used to verify entries for container and images objects:

```ruby
describe docker.object(container_id) do
  its('Configuration.Path') { should eq 'value' }
end
```

Be aware that docker object is very complex and the output varies by object. An example output in InSpec shell looks like:

```bash
inspec> docker.object('921f6363ceaa3c5253a31068a0c8cb7b3a78c1353cf6b4125c7a5460d4e8ce2a')
=> {"Id"=>"921f6363ceaa3c5253a31068a0c8cb7b3a78c1353cf6b4125c7a5460d4e8ce2a",
 "Created"=>"2017-04-29T11:27:45.846552618Z",
 "Path"=>"docker-entrypoint.sh",
 "Args"=>["postgres"],
 "State"=>
  {"Status"=>"running",
   "Running"=>true,
   "Paused"=>false,
   "Restarting"=>false,
   "OOMKilled"=>false,
   "Dead"=>false,
   "Pid"=>18850,
   "ExitCode"=>0,
   "Error"=>"",
   "StartedAt"=>"2017-04-29T11:27:46.414873644Z",
   "FinishedAt"=>"0001-01-01T00:00:00Z"},
 "Image"=>"sha256:390220867755ef4d1351705130960605d14a44518149e89c674189eefcb09306",
 "ResolvConfPath"=>"/var/lib/docker/containers/921f6363ceaa3c5253a31068a0c8cb7b3a78c1353cf6b4125c7a5460d4e8ce2a/resolv.conf",
 "HostnamePath"=>"/var/lib/docker/containers/921f6363ceaa3c5253a31068a0c8cb7b3a78c1353cf6b4125c7a5460d4e8ce2a/hostname",
 "HostsPath"=>"/var/lib/docker/containers/921f6363ceaa3c5253a31068a0c8cb7b3a78c1353cf6b4125c7a5460d4e8ce2a/hosts",
 "LogPath"=>"/var/lib/docker/containers/921f6363ceaa3c5253a31068a0c8cb7b3a78c1353cf6b4125c7a5460d4e8ce2a/921f6363ceaa3c5253a31068a0c8cb7b3a78c1353cf6b4125c7a5460d4e8ce2a-json.log",
 "Name"=>"/some-postgres",
 "AppArmorProfile"=>"",
 ````



## Verify secure configuration

If you are going to run your own docker environment in production, you should make the additional work to harden the docker environment. The [CIS Docker Benchmark](https://benchmarks.cisecurity.org/tools2/docker/CIS_Docker_1.13.0_Benchmark_v1.0.0.pdf) provides a detailed overview about all best-practices. Based on those guidelines, the [DevSec Hardening Framework Team](http://atomic111.github.io/blog/inspec-cis-docker) implemented a machine-readable version that allows you to easily execute the tests. To execute the InSpec profile, just run:

```bash
$ inspec exec https://github.com/dev-sec/cis-docker-benchmark
```

## Conclusion

This blog post covered various aspects of docker environment verification ranging from docker hosts to docker container. If you feel that something is missing, please let me know.

Stay secure, Chris


## References

- [InSpec docker resource](https://www.inspec.io/docs/reference/resources/docker/)
- [InSpec docker_container resource](https://www.inspec.io/docs/reference/resources/docker_container/)
- [InSpec docker_image resource](https://www.inspec.io/docs/reference/resources/docker_image/)
- [DevSec CIS Docker Benchmark Implementation](https://github.com/dev-sec/cis-docker-benchmark)
- [CIS Docker Benchmark PDF](https://benchmarks.cisecurity.org/tools2/docker/CIS_Docker_1.13.0_Benchmark_v1.0.0.pdf)
- [Docker Pull Request Resource](https://github.com/chef/inspec/pull/1566)
