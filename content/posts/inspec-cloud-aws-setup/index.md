---
title: Getting started with InSpec for AWS. Testing for the cloud!
author: Christoph Hartmann
date: 2018-03-28
tags:
  - inspec
  - aws
aliases:
  - /articles/inspec-cloud-aws-setup/
---

With the introduction of InSpec 2.0, we got the ability to test AWS environments. Within the next 5 minutes, you are ready to write InSpec tests to verify your AWS environment. Let's start.

## Background

Dominik already introduced the new concepts of [InSpec 2.0](https://thenewstack.io/moving-beyond-limits-infrastructure-testing-inspec-2-0/). He and I created InSpec for machine testing but abstracted InSpec's test syntax from specific test targets. This is very helpful, since the same language can be used to describe tests for machine and api testing. 

A simple test for an operating system package can be written like:

```ruby
# test for os package
describe package('telnetd') do
  it { should_not be_installed }
end
```

InSpec CLI can easily execute this test locally or remotely with:

```bash
# run test locally, local execution is implicit
inspec exec test.rb

# run test on docker container
inspec exec test.rb -t docker://container_id
```

A test to verify that a EC2 instance exists and is running is written as following:

```ruby
# test for aws ec2
describe aws_ec2_instance('i-01a2349e94458a507') do
  it { should exist }
  it { should be_running }
end
```

Now, we target the AWS api instead of local, ssh, winrm or docker: 

```bash
# run a profile targeting AWS using env vars
inspec exec test.rb -t aws://
```

While the first example is a test for an operating system, the second is a test for the AWS API. They follow the same concepts, you either target your tests to an operating system or an AWS API. InSpec is smart enough to know that a resource works for a given target.

{{< figure src="inspec-exec-wrong-platform.png" alt="InSpec detects the target platform" title="InSpec detects the target platform" >}}

In case you just want to play with InSpec for testing, I recommend using my `inspec-playgroud` container, since it includes InSpec and AWS CLI:

```bash
$ docker run -it chrisrock/inspec-playground
[root@b7a17c8c6dd4 /]# inspec version
2.1.10
[root@b7a17c8c6dd4 /]# aws --version
aws-cli/1.14.63 Python/2.7.5 Linux/4.9.60-linuxkit-aufs botocore/1.9.16
```

## Create an AWS IAM user

To allow InSpec to talk to AWS, we need to create an IAM user in AWS console. If you have credentials already, skip to the next [section](#run-inspec-check). The following commands are executed on Linux but work similar on Windows and macOS. If you've never run AWS cli before, you should see an error message like: 

```bash
[root@b7a17c8c6dd4 /]# aws iam get-user
Unable to locate credentials. You can configure credentials by running "aws configure".
```

Create new credentials within the [IAM console](https://console.aws.amazon.com/iam/home#/users) by clicking on "add user". Now enter your username and select `programmatic access`.

{{< figure src="iam_create_user_01.png" alt="Create a new user" title="Create a new user" >}}

As the second step, we need to provide permissions to the user. Since InSpec is for testing only, `ReadOnlyAccess` is all you need.

{{< figure src="iam_create_user_02.png" alt="Select IAM permissions" title="Select IAM permissions" >}}

Now, we need to confirm the configuration.

{{< figure src="iam_create_user_03.png" alt="Review configuration" title="Review configuration" >}}

If everything goes well, you should see the confirmation page with the AWS Access Key and AWS Secret Access Key. Both are required for the next step.

{{< figure src="iam_create_user_04.png" alt="IAM user creation confirmation" title="IAM user creation confirmation" >}}

## Configure your AWS CLI

The AWS cli makes it very easy to configure AWS settings. InSpec is reading the same configuration files, therefore the AWS CLI works hand-in-hand with InSpec. No seperate configuration required.

```bash
[root@b7a17c8c6dd4 /]#  aws configure
AWS Access Key ID [None]:  AKIAXXXXXXXXXXXXXXXXX
AWS Secret Access Key [None]: l1g3J+BAI1EWyyWR4c5n5xxxxxxxxxxxxxxxx
Default region name [None]: us-east-1
Default output format [None]:
```

Now, the `aws` cli created two config files:

```bash
[root@b7a17c8c6dd4 /]# ls ~/.aws/            
config  credentials
[root@b7a17c8c6dd4 /]# cat ~/.aws/config 
[default]
region = us-east-1
[root@b7a17c8c6dd4 /]# cat ~/.aws/credentials 
[default]
aws_access_key_id =  AKIAXXXXXXXXXXXXXXXXX
aws_secret_access_key = l1g3J+BAI1EWyyWR4c5n5xxxxxxxxxxxxxxxx
```

Verify that `aws iam get-user` can connect to AWS. You get a response, everything is configure properly.

```bash
[root@b7a17c8c6dd4 /]# aws iam get-user
{
    "User": {
        "UserName": "chris-rock", 
        "Path": "/", 
        "CreateDate": "2018-03-27T12:06:32Z", 
        "UserId": "AKIAXXXXXXXXXXXXXXXXX", 
        "Arn": "arn:aws:iam::112758395563:user/chris-rock"
    }
}
```

# Run first AWS InSpec check

InSpec does not rely on AWS cli. It re-uses the same configuration files located in `~/.aws/config` and `~/.aws/credentials` though. Since we configured them already, we are ready to go.

Let's create a new InSpec test file `iam.rb` with the following content

```ruby
describe aws_iam_user(username: 'chris-rock') do
    it { should have_mfa_enabled }
    it { should_not have_console_password }
end
```

Execute that test with InSpec:

```bash
$ inspec exec iam.rb -t aws://
```

And  voilà!

{{< figure src="inspec-aws-run.png" alt="InSpec reports DevSec SSH Baseline to Chef Automate" title="InSpec reports DevSec SSH Baseline to Chef Automate" >}}


If you are using multiple AWS profiles you can define those in your `~/.aws/credentials` file.

```ini
$ cat ~/.aws/credentials 
[auditing]
aws_access_key_id = AKIA....
aws_secret_access_key = 1234....abcd
```

You can use the AWS profile as an option for the inspec target `-t aws://region/profile`. For example, to connect to the Ohio region using a profile named `auditing`, use `inspec exec iam.rb -t aws://us-east-2/auditing`. Please have a look look at the [InSpec Docs](https://www.inspec.io/docs/reference/platforms/) for more information.

## Summary

Yeah! Within 5 minutes, we wrote our first InSpec test for AWS. InSpec ships with a lot of [built-in AWS resources](https://www.inspec.io/docs/reference/resources/#aws-resources). You may also be interested to combine [Terraform with InSpec](http://lollyrock.com/articles/inspec-terraform/) as the next step.

Happy Wednesday! - Chris

