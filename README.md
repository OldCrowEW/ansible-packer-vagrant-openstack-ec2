# ansible-packer-vagrant-openstack-ec2

Ansible scripts to build operating system images using Packer and Vagrant, 
exporting to an Openstack image, Amazon EC2 AMI or OVF image(s).

Now also offers the possibility to upload the image directly to Openstack
Image service (Glance) and/or automatically instanciate it with a
Heat orchestration template.

Please see the end of the document for the changelog, it contains some
important information regarding changes.

## Required software:

- Ansible (http://www.ansible.com/home)
- Packer (https://www.packer.io/), at least version 0.8.6 for ssh_pty support
- Vagrant (https://www.vagrantup.com/)
- qemu-img (available for OS X in Brew's qemu package)
- Boto for ec2_ami module (pip install boto)

These scripts assume that you are using Virtualbox to back Vagrant boxes.

## Required setup for OVF images

Change "build_ovf_image" to true (from command line or playbook vars).

The import has only been tested on Virtualbox.

## Required setup for AMI images

Change "build_ami_image" to true (from command line or playbook vars), this
builds an instance-store backed AMI. For EBS-backed images, set "build_ami_ebs_image" 
to true.

Also you'll need to set your AWS_ACCESS_KEY and AWS_SECRET_KEY:

```
export AWS_ACCESS_KEY=XXXXXXXXXXXXXXXXXX
export AWS_SECRET_KEY=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Then finally you'll need your client number (twelve digits from Amazon) and a X.509 
certificate from the IAM console (configure in the playbook build-image.yml vars 
section, amazon_private_key and amazon_certificate).

This image uses HVM virtualization.

## You'll also need:

- An ISO image you would like to install (this example uses CentOS 7), place 
  it in packer/ directory (place the iso image filename in build-image.yml variables).
  Example: 
  wget -Opacker/CentOS-7-x86_64-Minimal-1507-01.iso http://buildlogs.centos.org/rolling/7/isos/x86_64/CentOS-7-x86_64-Minimal-1507-01.iso
- Miscellaneous utilities (should come bundled with OS X): xmllint, tidy,
  openssl, perl, split, uuidgen

Tested on OS X 10.10, but should be easily adapted to Linux.

## Running the play

Simply run:
```
ansible-playbook -i hosts build-image.yml
```

Pro-tip: No Packer build (when you want to debug your scripts more quickly and you
already have a base Packer build):
```
ansible-playbook -i hosts build-image.yml -e 'no_packer_build=yes'
```

If everything was successful, you'll find the QCOW2 image in your Ansible directory.
If you built an AMI image, it should be available under your AMI images.

## Uploading to Openstack

Set build_glance_image to yes and setup openstack_image as you'd like. You'll
need the OS_* variables set up to communicate with Openstack API.

You can use os_image.py from your Ansible distribution or the one bundled
with this playbook.

## Heat orchestration

There's a simple example template in roles/make-heat-template/templates/sample.yml.j2
which starts a single server with the image you created. 

There's a new Ansible module called os_orchestration_stack.py which allows uploading
Heat templates programmatically.

## Change log:

### 9.8.2015
- Added option to generate EBS backed AMI images. This needs
  support in Boto, which has not yet been merged in:

  https://github.com/boto/boto/pull/3333 and
  https://github.com/boto/boto/pull/3332

  The ec2_ami module will check for compatibility.

### 8.8.2015:
- Removed requirement for Amazon's CLI tools, use s3 to upload image
  parts and improved ec2_ami module to register the image. This
  requires a small fix to Boto's canned ACL strings list at:
  boto/s3/acl.py 

```
--- acl.py.old	2015-09-08 12:53:26.000000000 +0300
+++ acl.py	2015-09-08 12:53:18.000000000 +0300
@@ -25,7 +25,7 @@
 CannedACLStrings = ['private', 'public-read',
                     'public-read-write', 'authenticated-read',
                     'bucket-owner-read', 'bucket-owner-full-control',
-                    'log-delivery-write']
+                    'log-delivery-write', 'aws-exec-read']
```

  Boto issue: https://github.com/boto/boto/issues/3330

  This also requires a small patches to s3 and ec2_ami Ansible modules,
  which are both included. 