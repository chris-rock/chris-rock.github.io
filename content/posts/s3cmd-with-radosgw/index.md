---
title: s3cmd with radosgw
author: chris
date: 2014-09-07
template: article.jade
---

Amazon introduced the concept of [S3](http://en.wikipedia.org/wiki/Amazon_S3) object storage to a wide-range of users. Their interface is the defacto-standard to store files in web applications. Nowadays, it is used by other vendors as well. [Ceph](http://ceph.com/) and [RiakCS](http://basho.com/riak-cloud-storage/) are some examples, where the same interface is available. This blog post will setup s3cmd with Ceph radosgw.

## About S3

It is used as an interface for distributed storage due to the fact that the only thing you need to put and retrieve files is http. It is very handy to have a clear interface to manage files across different machines. For example you may want to backup your local files to a s3 cluster. This requires a toolset that enables you to easily store and retrieve s3 files from command line. [s3cmd](http://s3tools.org/s3cmd) is a great tool for just that. Out of the box it works with Amazon S3, but can be easily configured with [radosgw](http://ceph.com/docs/master/man/8/radosgw/), which is commonly used in conjunction with [OpenStack](http://www.openstack.org/).

## Installation

Before we start implementing the base set, you need to install `s3cmd` on you machine. 

### Ubuntu 

For Ubuntu you need to install the s3 repo before installing the package `s3cmd`:

```bash
# install repo key
wget -O- -q http://s3tools.org/repo/deb-all/stable/s3tools.key | sudo apt-key add -
# install repo
sudo wget -O/etc/apt/sources.list.d/s3tools.list http://s3tools.org/repo/deb-all/stable/s3tools.list
# update package manager
sudo apt-get update
# install s3cmd
sudo apt-get install s3cmd
```

### MacOS

On Mac you may use [brew](http://brew.sh/) to install s3cmd:

```
# install s3cmd
brew install s3cmd
```

## Basic Commands

S3 uses a different terminology, compared to posix filesystems:

- bucket: kind of directory
- object: similar to a file

Although Amazon S3 allows you to create a `directory` within a bucket, it will not be a real directory. In S3, a directory is more like a file prefix. Take this in consideration when you upload huge amounts of data.

In general it is very efficient to use prefixes for searches instead of postfixes. E.g. if you look for file types, you may store your files as `txt.file` instead of `file.txt`.

Basic commands for `s3cmd` are:

```bash
# S3cmd help
s3cmd --help

# Display all buckets
s3cmd ls

# Create new bucket
s3cmd mb s3://YOURBUCKET

# Upload new file
s3cmd put test.txt s3://YOURBUCKET

# Download file
s3cmd get s3://YOURBUCKET/test.txt test.txt
```

## Configuration for radosgw

Since `s3cmd` is optimized for Amazon S3, you need to apply a few changes to convince `s3cmd` to talk with `radosgw`. At first we create a new configuration with s3cmd via interactive mode:

```code
s3cmd --configure -c s3test.cfg

Enter new values or accept defaults in brackets with Enter.
Refer to user manual for detailed description of all options.

Access key and Secret key are your identifiers for Amazon S3
Access Key: key
Secret Key: secret

Encryption password is used to protect your files from reading
by unauthorized persons while in transfer to S3
Encryption password: 
Path to GPG program [/usr/local/bin/gpg]: 

When using secure HTTPS protocol all communication with Amazon S3
servers is protected from 3rd party eavesdropping. This method is
slower than plain HTTP and can't be used if you're behind a proxy
Use HTTPS protocol [No]: yes

New settings:
  Access Key: key
  Secret Key: secret
  Encryption password: 
  Path to GPG program: /usr/local/bin/gpg
  Use HTTPS protocol: True
  HTTP Proxy server name: 
  HTTP Proxy server port: 0

Test access with supplied credentials? [Y/n] n

Save settings? [y/N] y
Configuration saved to 's3test.cfg'
```

The generated configuration file will look similar to:

```code
[default]
access_key = abc
bucket_location = US
cloudfront_host = cloudfront.amazonaws.com
cloudfront_resource = /2010-07-15/distribution
default_mime_type = binary/octet-stream
delete_removed = False
dry_run = False
encoding = UTF-8
encrypt = False
follow_symlinks = False
force = False
get_continue = False
gpg_command = /usr/local/bin/gpg
gpg_decrypt = %(gpg_command)s -d --verbose --no-use-agent --batch --yes --passphrase-fd %(passphrase_fd)s -o %(output_file)s %(input_file)s
gpg_encrypt = %(gpg_command)s -c --verbose --no-use-agent --batch --yes --passphrase-fd %(passphrase_fd)s -o %(output_file)s %(input_file)s
gpg_passphrase =
guess_mime_type = True
host_base = s3.amazonaws.com
host_bucket = %(bucket)s.s3.amazonaws.com
human_readable_sizes = False
list_md5 = False
log_target_prefix =
preserve_attrs = True
progress_meter = True
proxy_host =
proxy_port = 0
recursive = False
recv_chunk = 4096
reduced_redundancy = False
secret_key = secret
send_chunk = 4096
simpledb_host = sdb.amazonaws.com
skip_existing = False
socket_timeout = 300
urlencoding_mode = normal
use_https = True
verbosity = WARNING
```

Afterwards you may try to access your bucket with `s3cmd`:

```bash
s3cmd -c s3test.cfg ls
ERROR: S3 error: 403 (InvalidAccessKeyId): The AWS Access Key Id you provided does not exist in our records.
```

We need to adapt a few lines to configure s3cmd for radosgw. Currently `s3cmd` does not know the radosgw hostname. Replace `host_base` and `host_bucket` with the hostname of the radosgw host:

```code
host_base = s3.amazonaws.com
host_bucket = %(bucket)s.s3.amazonaws.com
```

becomes:

```code
host_base = radosgw.example.com
host_bucket = %(bucket)s.radosgw.example.com
```

Now you should be able to run 

```code
s3cmd -c s3test.cfg ls
2014-09-06 12:05  s3://YOURBUCKET
```

Sidenote: I had a few issues using lowercase bucket names with `radosgw`. Uppercase buckets worked like a charm. 


## Encryption with gpg

If we talk about backups, we never want to store them unencrypted. `s3cmd` supports [gpg](https://www.gnupg.org/). The cool thing of using s3cmd in conjunction with gpg is the fact that it works completely transparent. Once configured, all files are automatically encrypted during upload and decrypted whenever you download a file.

To activate encryption change the `s3cmd` configuration file:

```code
encrypt = True
gpg_command = gpg
gpg_decrypt = %(gpg_command)s -d --verbose --no-use-agent --batch --yes --passphrase-fd %(passphrase_fd)s -o %(output_file)s %(input_file)s
gpg_encrypt = %(gpg_command)s -c --cipher-algo AES256 --force-mdc --verbose --no-use-agent --batch --yes --passphrase-fd %(passphrase_fd)s -o %(output_file)s %(input_file)s
gpg_passphrase = longsecurepassphrase
```

At first you have to enable encryption with `encrypt = True`. Although the default configuration uses `gpg_command = /usr/local/bin/gpg`, I prefer to use `gpg_command = gpg`. Now, you have to set `gpg` in your system path, but on the other hand, the configuration works on Linux and Mac without any change. Additionally you want to ensure that gpg uses the right cipher. I prefer to set the cipher explicitly: `--cipher-algo AES256`. Although default with AES256, the option `--force-mdc` forces the use of encryption with a modification detection code.

Finally, `s3cmd` works with `radosgw` and encryption.

```
$> cat > test.txt
samplecontent
$> s3cmd put test.txt s3://YOURBUCKET
test.txt -> s3://YOURBUCKET/test.txt  [1 of 1]
 4 of 4   100% in    0s     7.51 B/s  done
$> s3cmd get s3://YOURBUCKET/test.txt test2.txt
s3://YOURBUCKET/test.txt -> test2.txt  [1 of 1]
 4 of 4   100% in    0s     5.74 B/s  done
$> s3cmd del s3://YOURBUCKET/test.txt
File s3://YOURBUCKET/test.txt deleted
$> cat test2.txt 
samplecontent

```

If you have any questions contact me via [Twitter @chri_hartmann](https://twitter.com/chri_hartmann) or [Github](https://github.com/chris-rock)
