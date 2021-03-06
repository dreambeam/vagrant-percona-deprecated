# Vagrant + Percona 

## Introduction

This repository contains tools to build consistent environments for testing Percona software on a variety of platforms.  This includes EC2 and Virtualbox for now, but more are possible going forward.

Principles/goals of this environment:

* Extremely Reusable
* Small manifests to be used by multiple vagrant providers to combine components for needed boxes
* Vagrantfiles are very descriptive about the whole environment needed.  Preference given to making modules configurable rather than custom. 
* Useful for:
 * Conference tutorial environments
 * Training classes
 * Experimentation
 * Benchmarking
* Manifest install categories:
 * MySQL and variants
 * MySQL tools
 * Benchmarking tools
 * Sample databases
 * Misc: local repos for conference VMs, 

## Walkthrough

This section should get you up and running.

### Software Requirements

* Vagrant 1.6+: http://vagrantup.com
* Vagrant AWS Plugin (optional):

```
 vagrant plugin install vagrant-aws
```

* VirtualBox: https://www.virtualbox.org (optional)
* VMware Fusion (not supported yet, but feasible)
* Vagrant Host Manager Plugin

```
 vagrant plugin install vagrant-hostmanager
```

### Openstack Setup

For this to run, you'll need a custom Vagrantbox with an image to boot from on your Openstack Cloud.  I've created a CentOS one here: https://github.com/grypyrg/packer-percona, but it would need to be rebuilt in other clouds.

Perconians can use a prebuilt image in our Openstack lab with this command: 

```
vagrant box add grypyrg/centos-x86_64 --provider openstack
```

You'll also need your secrets setup in ~/.openstack_secrets:

```yaml
---
endpoint: http://controller:5000/v2.0/tokens
tenant: tenant_name
username: your_user
password: your_pw
keypair_name: your_keypair_name
private_key_path: the_path_to_your_pem_file
```

Finally, you'll need the vagrant-openstack-plugin:

```
vagrant plugin install vagrant-openstack-plugin
```

### AWS Setup

You can skip this section if you aren't planning on using AWS.  

In a nutshell, you need this:

* AWS access key
* AWS secret access key
* A Keypair name and path for each AWS region you intend to use
* Whatever security groups you'll need for the environments you intend to launch.

#### AWS Details

You'll need an AWS account setup with the following information in a file called ~/.aws_secrets:

```yaml
access_key_id: YOUR_ACCESS_KEY
secret_access_key: THE_ASSOCIATED_SECRET_KEY
keypair_name: KEYPAIR_ID
keypair_path: PATH_TO_KEYPAIR_PEM
instance_name_prefix: SOME_NAME_PREFIX
default_vpc_subnet_id: subnet-896602d0
```

#### Multi-region

AWS Multi-region can be supported by adding a 'regions' hash to the .aws_secrets file:

```yaml
access_key_id: YOUR_ACCESS_KEY
secret_access_key: THE_ASSOCIATED_SECRET_KEY
keypair_name: jay
keypair_path: /Users/jayj/.ssh/jay-us-east-1.pem
instance_name_prefix: Jay
default_vpc_subnet_id: subnet-896602d0
regions:
  us-east-1:
    keypair_name: jay
    keypair_path: /Users/jayj/.ssh/jay-us-east-1.pem
    default_vpc_subnet_id: subnet-896602d0
  us-west-1:
    keypair_name: jay
    keypair_path: /Users/jayj/.ssh/jay-us-west-1.pem
  eu-west-1:
    keypair_name: jay
    keypair_path: /Users/jayj/.ssh/jay-eu-west-1.pem
```

Note that the default 'keypair_name' and 'keypair_path' can still be used.  Region will default to 'us-east-1' unless you specifically override it.    

#### Boxes and Multiple AWS Regions

AMI's are region-specific. The AWS Vagrant boxes you use must include AMI's for each region in which you wish to deploy.

For an example, see the regions listed here: https://vagrantcloud.com/grypyrg/centos-x86_64.

Packer, which is used to build this box, can be configured to add more regions if desired, but it requires building a new box.

#### AWS VPC Integration

The latest versions of grypyrg/centos-x86-64 boxes require a VPC since AWS now requires VPC for all instances. 

As shown in the example above, you must set the `default_vpc_subnet_id` in the ~/.aws_secrets file. You can override this on a per-region basis.

You can also pass a `subnet_id` into the `provider_aws` method using an override in your Vagrantfile.

### Clone this repo

```bash
git clone <clone URL> 
cd vagrant-percona
git submodule init
git submodule update --recursive
```

### Launch the box

Launch your first box -- ps_sysbench is a good start.  

```bash
ln -sf Vagrantfile.ps_sysbench.rb Vagrantfile
vagrant up
vagrant ssh
```

### Create Environments with create-new-env.sh

When you create a lot of vagrant environments with vagrant-percona, creating/renaming those Vagrantfile files can get quite messy easily.

The repository contains a small script that allows you to create a new environment, which will build a new directory with the proper Vagrantfile files and links to the puppet code.

This allows you to have many many Vagrant environments configured simultaneously.

```bash
vagrant-percona$ ./create-new-env.sh single_node ~/vagrant/testing-issue-428
Creating 'single_node' Environment

vagrant-percona$ cd ~/vagrant/testing-issue-428
~/vagrant/testing-issue-428$ vagrant up --provider=aws
~/vagrant/testing-issue-428$ vagrant ssh
```

## Master/Slave

This Vagrantfile will launch 2 (or more; edit the file and uncomment proper build line) MySQL servers in either VirtualBox or AWS. Running the ms-setup.pl script will set the first instance to be the master and all remaining nodes to be async slaves.

```bash
ln -sf Vagrantfile.ms.rb Vagrantfile
vagrant up --provider [aws|virtualbox]
./ms-setup.pl
```  

## PXC 

This Vagrantfile will launch 3 Percona 5.7 XtraDB Cluster nodes in either VirtualBox or AWS. The InnoDB Buffer Pool is set to 128MB. The first node is automatically bootstrapped to form the cluster. The remaining 2 nodes will join the first to form the cluster.

Each Virtualbox instance is launched with 256MB of memory.

Each EC2 instance will use the `m3.medium` instance type, which has 3.75GB of RAM.

```bash
ln -sf Vagrantfile.pxc.rb Vagrantfile
vagrant up
```  

__NOTE:__ Due to Vagrant being able to parallel build in AWS, there is no guarantee "node 1" will bootstrap before the other 2. If this happens, node 2 and node 3 will be unable to join the cluster. It is therfore recommended you launch node 1 manually, first, then launch the remaining nodes. _(This is not an issue with Virtualbox as parallel builds are not supported.)_

Example:

```bash
vagrant up node1 && sleep 5 && vagrant up
```

## PXC (Big)

This Vagrantfile will launch 3 Percona 5.7 XtraDB Cluster nodes in either VirtualBox or AWS. The InnoDB Buffer Pool is set to _12GB_. 

__WARNING:__ This requires a virtual machine with 15GB of RAM. Most consumer laptops and desktops do not have the RAM requirements to run multiple nodes of this configuration.

Each EC2 instance will use the `m3.xlarge` instance type, which has 15GB of RAM.

```bash
ln -sf Vagrantfile.pxc-big.rb Vagrantfile
vagrant up
```

__NOTE:__ Due to Vagrant being able to parallel build in AWS, there is no guarantee "node 1" will bootstrap before the other 2. If this happens, node 2 and node 3 will be unable to join the cluster. It is therfore recommended you launch node 1 manually, first, then launch the remaining nodes. _(This is not an issue with Virtualbox as parallel builds are not supported.)_

Example:

```bash
vagrant up node1 && sleep 5 && vagrant up
```

## Using this repo to create benchmarks

I use a system where I define this repo as a submodule in a test-specific git repo and do all the customization for the test there.

```bash
git init some-test
cd some-test
git submodule add git@github.com:grypyrg/vagrant-percona.git
ln -s vagrant-percona/lib
ln -s vagrant-percona/manifests
ln -s vagrant-percona/modules
cp vagrant-percona/Vagrantfile.of_your_choice Vagrantfile
vi Vagrantfile  # customize for your test
vagrant up
...
```

## Cleanup

### Shutdown the vagrant instance(s)

```
vagrant destroy
```
