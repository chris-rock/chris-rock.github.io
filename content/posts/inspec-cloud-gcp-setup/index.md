---
title: Google Cloud Platform support for InSpec
author: Christoph Hartmann
date: 2018-05-23
tags:
  - inspec
  - gcp
aliases:
  - /articles/inspec-cloud-gcp-setup/
---

When we released [InSpec 2.0](https://techcrunch.com/2018/02/20/chef-inspec-2-0-wants-to-help-companies-automate-security-compliance-in-cloud-apps/) in February 2018, it shipped with native support for AWS and Azure. Over the course of the last 3 months, the InSpec team and community kept adding more AWS and Azure resources. We also showcased how [Terraform can be tested effectively with InSpec](http://lollyrock.com/articles/inspec-terraform/). In parallel, we worked on Google Cloud Platform (GCP) support which is now available.

## Verify Google Cloud Platform resources

{{< figure src="inspec-gcp.png" alt="Google Cloud Platform support for InSpec" title="Google Cloud Platform support for InSpec" >}}

We wanted to extend the cloud support for a while and the InSpec community has been super fast in its adoption! [Martez Reed](https://github.com/martezr/inspec-gcp) pushed an early-prototype right after InSpec 2.0 has been announced and [Thomas Poindessous](https://github.com/chef/inspec/issues/2722) brought up the needs for GCP support in the community. We've listened and worked with Google to get GCP support into InSpec.

To get native GCP support, we added [GCP support in Train](https://github.com/chef/train/pull/283) and worked on [InSpec GCP Resources](https://github.com/inspec/inspec-gcp). The resource pack is like an InSpec profile with custom resources but without any controls. This resource pack allows us to improve the stability and documentation of the resources before we bring them into core InSpec. [Stuart Paterson](https://github.com/skpaterson) built those resources on top of Martez's ideas. Now, let's see how this works in practice.

## Preparation

InSpec GCP resources require a GCP client ID and secret. The easiest way to set up the credentials is via the Google SDK. Please install and configure the GCP SDK by downloading the [SDK](https://cloud.google.com/sdk/docs/) and running the installation via `./google-cloud-sdk/install.sh`. Once everything is installed, we are ready to gather the credentials: 

```bash
gcloud auth application-default login
cat ~/.config/gcloud/application_default_credentials.json 
{
  "client_id": "764086051850-6qp4p6gpi6in60asdr.apps.googleusercontent.com",
  "client_secret": "d-fasdabcdabcdabceroi123knrmfs;fabc",
  "refresh_token": "1/asdfjlklabc;ldabc'dfmk-lCkju3-yQmjr20xVZabcfkE48L",
  "type": "authorized_user"
}
```

If you are a fist time GCP user, you may be required to enable the GCP APIs like [Compute Engine API](https://console.cloud.google.com/apis/library/compute.googleapis.com/) or [Kubernetes Engine API](https://console.cloud.google.com/apis/library/container.googleapis.com).


Since GCP setup is ready, please update to the latest [InSpec](https://www.inspec.io/downloads/) version. At least version 2.1.78 is required. To verify that the access works, run:

```bash
$ inspec detect -t gcp://

== Platform Details

Name:      gcp
Families:  cloud, api
Release:   google-cloud-v

```

## Create a new GCP profile

To create a new GCP profile, use `inspec init profile` command:

```bash
$ inspec init profile gcp-example-profile
Create new profile at /Users/chris-rock/gcp-example-profile
 * Create directory libraries
 * Create file README.md
 * Create directory controls
 * Create file controls/example.rb
 * Create file inspec.yml
 * Create file libraries/.gitkeep
$ cd gcp-example-profile
$ cat inspec.yml
name: gcp-example-profile
title: InSpec Profile
maintainer: The Authors
copyright: The Authors
copyright_email: you@example.com
license: Apache-2.0
summary: An InSpec Compliance Profile
version: 0.1.0
```

InSpec creates the profile structure that looks as following:

```bash
$ tree .
.
├── README.md
├── controls
│   └── example.rb
├── inspec.yml
└── libraries

2 directories, 3 files
```

The `inspec.yml` contains the profile metadata, `example.rb` includes the InSpec tests. Now, we adapt the `inspec.yml` to load the InSpec resource pack for Google Cloud Platform and tell InSpec that the profile is intended for GCP.

```yaml
name: gcp-example-profile
...
depends:
  - name: gcp-resources
    url: https://github.com/inspec/inspec-gcp/archive/master.tar.gz
supports:
  - platform: gcp
```

Now, we edit the `example.rb` verify that our GCP project exists via the `google_project` InSpec resource:

```ruby
title 'sample gcp test section'

PROJECT_NUMBER = attribute('project_number', description: 'gcp project number')

control 'gcp-1' do
  impact 0.7
  title 'Check development project'
  describe google_project(project: PROJECT_NUMBER) do
    it { should exist }
    its('name') { should eq 'inspec-gcp' }
    its('project_number') { should cmp PROJECT_NUMBER }
    its('lifecycle_state') { should eq 'ACTIVE' }
  end
end
```

We also use InSpec [attributes](https://www.inspec.io/docs/reference/profiles/#profile-attributes) to make the profile more flexible. Just create an `attributes.yml` that includes the `project_number` value

```yaml
project_number: 41681219238
```

Now, run the profile with InSpec:

```bash
inspec exec . -t gcp:// --attrs attributes.yml

Profile: InSpec Profile (gcp-example-profile)
Version: 0.1.0
Target:  gcp://764086051850-6qr4p6gpi6hn506pt8ejuq83di341hur.apps.googleusercontent.com

  ✔  gcp-1: Check development project
     ✔  Project  should exist
     ✔  Project  name should eq "inspec-gcp"
     ✔  Project  project_number should cmp == 41681219238
     ✔  Project  lifecycle_state should eq "ACTIVE"


Profile: Google Cloud Platform Resource Pack (inspec-gcp)
Version: 0.2.0
Target:  gcp://764086051850-6qr4p6gpi6hn506pt8ejuq83di341hur.apps.googleusercontent.com

     No tests executed.

Profile Summary: 1 successful control, 0 control failures, 0 controls skipped
Test Summary: 4 successful, 0 failures, 0 skipped

```

As the next step, we add a test to verify an google storage bucket by using the `google_storage_bucket` resource:

```ruby
BUCKET = attribute('storage_bucket', description: 'gcp storage bucket identifier')

control 'gcp-3' do
  impact 0.3
  title 'Check that the storage bucket was created'
  describe google_storage_bucket(name: BUCKET) do
    it { should exist }
    its('storage_class') { should eq 'STANDARD' }
    its('location') { should eq 'EUROPE-WEST2' }
  end
end
```

Now, just update the `attributes.yml`:

```yaml
project_number: 41681219238
storage_bucket: gcp-inspec-storage-bucket-rgjqbngjzyeofzh
```

and run the profile with InSpec again:

```bash
inspec exec . -t gcp:// --attrs attributes.yml

Profile: InSpec Profile (gcp-example-profile)
Version: 0.1.0
Target:  gcp://764086051850-6qr4p6gpi6hn506pt8ejuq83di341hur.apps.googleusercontent.com

  ✔  gcp-1: Check development project
     ✔  Project  should exist
     ✔  Project  name should eq "inspec-gcp"
     ✔  Project  project_number should cmp == 41681219238
     ✔  Project  lifecycle_state should eq "ACTIVE"
  ✔  gcp-3: Check that the storage bucket was created
     ✔  Bucket gcp-inspec-storage-bucket-rgjqbngjzyeofzh should exist
     ✔  Bucket gcp-inspec-storage-bucket-rgjqbngjzyeofzh storage_class should eq "STANDARD"
     ✔  Bucket gcp-inspec-storage-bucket-rgjqbngjzyeofzh location should eq "EUROPE-WEST2"


Profile: Google Cloud Platform Resource Pack (inspec-gcp)
Version: 0.2.0
Target:  gcp://764086051850-6qr4p6gpi6hn506pt8ejuq83di341hur.apps.googleusercontent.com

     No tests executed.

Profile Summary: 2 successful controls, 0 control failures, 0 controls skipped
Test Summary: 7 successful, 0 failures, 0 skipped
```

As the final step, we verify a GCP instance via the `google_compute_instance` InSpec resource: 

```ruby
INSTANCE_NAME = attribute('instance_name', description: 'gcp instance identifier')
ZONE = attribute('instance_zone', description: 'instance zone')

control 'gcp-3' do
  impact 0.5
  title 'Check the GCP instance'
  describe google_compute_instance(project: PROJECT_NUMBER, zone: ZONE, name: INSTANCE_NAME) do
    it { should exist }
    its('name') { should eq 'gcp-inspec-int-linux-vm' }
    its('machine_type') { should eq 'f1-micro' }
    its('cpu_platform') { should match 'Intel' }
    its('status') { should eq 'RUNNING' }
  end
end
```

Since we use two additional attributes, we update the `attributes.yml` accordingly:

```yaml
project_number: 41681219238
storage_bucket: gcp-inspec-storage-bucket-rgjqbngjzyeofzh
instance_zone: europe-west2-a
instance_name: gcp-inspec-int-linux-vm2
```

Let's run the profile with InSpec again:

```bash
archlinux:..gcp-example-profile ±> inspec exec . -t gcp:// --attrs attributes.yml

Profile: InSpec Profile (gcp-example-profile)
Version: 0.1.0
Target:  gcp://764086051850-6qr4p6gpi6hn506pt8ejuq83di341hur.apps.googleusercontent.com

  ✔  gcp-1: Check development project
     ✔  Project  should exist
     ✔  Project  name should eq "inspec-gcp"
     ✔  Project  project_number should cmp == 41681219238
     ✔  Project  lifecycle_state should eq "ACTIVE"
  ✔  gcp-2: Check that the storage bucket was created
     ✔  Bucket gcp-inspec-storage-bucket-rgjqbngjzyeofzh should exist
     ✔  Bucket gcp-inspec-storage-bucket-rgjqbngjzyeofzh storage_class should eq "STANDARD"
     ✔  Bucket gcp-inspec-storage-bucket-rgjqbngjzyeofzh location should eq "EUROPE-WEST2"
  ✔  gcp-3: Check the GCP instance
     ✔  Instance gcp-inspec-int-linux-vm should exist
     ✔  Instance gcp-inspec-int-linux-vm name should eq "gcp-inspec-int-linux-vm"
     ✔  Instance gcp-inspec-int-linux-vm cpu_platform should match "Intel"
     ✔  Instance gcp-inspec-int-linux-vm status should eq "RUNNING"


Profile: Google Cloud Platform Resource Pack (inspec-gcp)
Version: 0.2.0
Target:  gcp://764086051850-6qr4p6gpi6hn506pt8ejuq83di341hur.apps.googleusercontent.com

     No tests executed.

Profile Summary: 3 successful controls, 0 control failures, 0 controls skipped
Test Summary: 11 successful, 0 failures, 0 skipped

```

## Summary

In just a few minutes, we created a few tests that verified some aspects of our GCP setup. This example profile is available at [Github](https://github.com/chris-rock/inspec-verify-provision) and should give you a good start into InSpec GCP. The InSpec GCP resource pack already ships with many [resources](https://github.com/inspec/inspec-gcp/tree/master/libraries):

 - `google_compute_address`
 - `google_compute_firewall`
 - `google_compute_image`
 - `google_compute_instance`
 - `google_compute_instance_group`
 - `google_container_cluster`
 - `google_container_node_pool`
 - `google_project.rb`
 - `google_project_iam_custom_role`
 - `google_service_account`
 - `google_storage_bucket`

We are always looking for more contributions and feedback for the InSpec resources. Please try out the new GCP resources and help us to make them better [questions or ideas](https://github.com/inspec/inspec-gcp/issues/new).

I am looking for your feedback!

- Chris 
