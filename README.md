# Ansible Plays for Typesafe ConductR

These plays and playbooks provision [Typesafe ConductR](https://conductr.typesafe.com) cluster nodes with [Ansible](http://www.ansible.com).

The playbook build-cluster-ec2.yml launches three instances, each in a different availibility zone. Conductr is installed on the instances and configured to form a cluster. The nodes are registered with a load balancer.

## Prerequisites

You'll need a few items in order to use these plays.

* Access Keys and Secrets for an AWS Account with permissions to admin EC2.
* Ansible installed on a controller host. The faster the controller host's connection to AWS, the faster nodes will launch. Ssh access to a small to medium instance in the same AWS region as your cluster works well. See Ansible Setup below for further details.
* An AWS Key Pair (PEM file) downloaded to the controller host.
* A copy of the ConductR deb installation package.
* A copy of this GitHub repo on the Ansible controller host.

## Setup

ConductR is not provided by this repository. [Contact Typesafe](http://www.typesafe.com/company/contact) to start your ConductR trial.

Copy the ConductR deb installation package into the `conductr/files` folder in your local copy of this repo. The installation package will be uploaded from this folder by the ConductR play to each of the EC2 instances for installation. 

Log into the AWS Console and select or generate a key pair. We'll need both the path to a local copy of the PEM file and the key pair name use in console to recored in our vars file.

Create the VPC subnets, security groups and load balancer as described in [Install-EC2](http://conductr.typesafe.com/intro/Install-EC2.html). Specify the keypair, load balancer, security-group and subnet names in the `vars/vars.yml`. Or wait for the next playbook to do this for you.

Populate `vars/vars.yml` with the names of your keypair, load balancer, security group and subnets.

## Running the Plays

From an Ansible enabled shell, export your AWS Key and Secret for use by the plays.

```bash
export AWS_ACCESS_KEY_ID='ABC123'
export AWS_SECRET_ACCESS_KEY='abc123'
```

Disable Ansible host key checking so that the play doesn't need you to accept the first connection to the nodes.

```bash
export ANSIBLE_HOST_KEY_CHECKING=False
```
We pass both our vars file and EC2 PEM key to our playbook as command line arguments. The VARS_FILE template must be populated with values from setup, above. The private-key value must be the local path and filename of the keypair that has the key pair name `KEYPAIR` specified in our vars file. For example our key pair may be named `ConductR_Key` in AWS and reside locally as `~/secrets/ConductR.pem`. In which case we would set `KEYPAIR` to `ConductR_Key` and pass `~/secrets/ConductR.pem` as our private-key argument.

All the nodes will get a public ip address so you can ssh into it using the same key using the user name from REMOTE_USER in `vars/main.yml`. 

```bash
export AWS_ACCESS_KEY_ID='ABC123'
export AWS_SECRET_ACCESS_KEY='abc123'
export ANSIBLE_HOST_KEY_CHECKING=False
ansible-playbook build-cluster-ec2.yml -e "VARS_FILE=vars/vars.yml" --private-key /path/to/{{keypair}}
```

Note: These plays use Oracle Java. If you cannot accept the Oracle license, use OpenJDK instead.

## Ansible Setup

These plays are being developed out of the current master branch of Ansible. They may or may not work with older packaged versions.

Setup Ansible from source using git and pip.

```bash
git clone https://github.com/ansible/ansible.git --recursive
sudo apt-get install python-setuptools autoconf g++ python2.7-dev
sudo easy_install pip
sudo pip install paramiko PyYAML Jinja2 httplib2 boto
```

Create a hosts file for Ansible.

```bash
sudo mkdir /etc/ansible
echo -e "[local]\n127.0.0.1" | sudo tee -a /etc/ansible/hosts
```

Configure a shell to use ansible from source

```bash
cd ansible
source ./hacking/env-setup -q
```
