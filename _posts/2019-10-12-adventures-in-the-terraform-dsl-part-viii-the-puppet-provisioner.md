---
layout: post
title: "Adventures in the Terraform DSL, Part VIII: The Puppet provisioner"
date: 2019-10-12
author: Alex Harvey
category: puppet
tags: terraform puppet
---

This post discussed a proof of concept of the Terraform 0.12.2 Puppet provisioner.

- ToC
{:toc}

## Introduction

In Terraform 0.12.2 a "basic Puppet provisioner" was added per feature request [#18851](https://github.com/hashicorp/terraform/pull/18851). The motivation for the provisioner is apparently to simplify installing, configuring and running Puppet Agents. And, since I am interested in both Terraform and Puppet, I decided to have a go at setting it up and doing a simple "hello world" with it. Also, I am fairly stubborn, and I even got it to work. This is the story of how I did it!

## Target audience

The post should help Puppet users who want to use the Terraform Puppet provisioner but it probably won't help Terraform users much with Puppet. I assume the reader has a good understanding of Puppet, Puppet Bolt and Terraform.

## The code

For readers who prefer to just go straight to the code, I have that all on GitHub [here](https://github.com/alexharv074/terraform-puppet-provisioner-test).

## Architecture

The following diagram shows the main moving parts of the solution:

![Puppet Terraform architecture]({{ "assets/arch.jpg" | absolute_url }})

Here is a bit about some of these:

|component|notes|
|---------|-----|
|Puppet Master<sup>1</sup>|The Puppet Master a.k.a. Puppet Server in today's Puppet. This is an open source Puppet 6 Puppet Server with the Autosign Ruby Gem installed.|
|Puppet Agent|The Puppet Agent node. In fact I have two of these, an Amazon Linux 2 Puppet Agent node and a Windows 2012 Puppet Agent node. It is here with installing and running Puppet that the Puppet Provisioner assists.|
|Puppet Bolt|Puppet Bolt is required to be on the machine running Terraform by the Puppet provisioner. Puppet Bolt tasks are called to autosign certificates on the Puppet Master and install Puppet on the Puppet Agent.|
|danieldreier/autosign|A Puppet module used by Puppet Bolt for autosigning Puppet agent Certificate Signing Requests. This and the following module is a dependency of the Terraform Puppet provisioner. They are managed outside of Terraform by Puppet Bolt and installed using bolt puppetfile install.|
|puppetlabs/puppet_agent|A Puppet module used by Puppet Bolt for managing Puppet Agent configuration.|

## Overview

The proof of concept code spins up a Puppet Master node, configures it using a UserData shell script, and then spins up an Amazon Linux 2 agent and a Windows 2012 agent in parallel and uses the Puppet provisioner to configure them both. And by "configure" I really just mean a simple Puppet manifest that prints "hello world" in the log. Why Windows 2012? That's what I found in Tim Sharpe's (the provisioner author's) [test code](https://github.com/rodjek/terraform-puppet-example).

Under the hood, the Terraform Puppet provisioner calls Puppet Bolt twice, once to sign the certificate signing request on the Puppet Master as the agent comes up for the first time and a second time to install the Puppet agent software on the node.

## Usage

### Setting up Puppet Bolt

Perhaps the most surprising feature of the Terraform Puppet provisioner is the requirement to have Puppet Bolt already set up on the machine where you run Terraform. So the first thing my code does is provide a simple shell script called setup.sh that installs and configures Puppet Bolt and then installs the Bolt Modules. (It assumes Mac OS X.) Here is that script:

```bash
#!/usr/bin/env bash

if ! command -v bolt ; then
  brew cask install puppetlabs/puppet/puppet-bolt
fi

mkdir -p ~/.puppetlabs/bolt/

(cd bolt && cp \
    bolt.yaml \
    Puppetfile \
    ~/.puppetlabs/bolt/)

bolt puppetfile install
```

This is fairly self-explanatory.

### Bolt config

#### bolt.yaml

The bolt.yaml meanwhile has this content:

```yaml
---
modulepath: "~/.puppetlabs/bolt-code/modules:~/.puppetlabs/bolt-code/site-modules"
concurrency: 10
format: human
ssh:
  host-key-check: false
  user: ec2-user
  private-key: ~/.ssh/default.pem
```

As the reader will observe below, I have had to specify some of these SSH connection details twice - here, and also in Terraform. The inconsistency seems to be that I have to tell both Terraform and Bolt about the SSH user ec2-user but I can only tell Bolt about the private key here.

### Puppetfile contents

I will also say something about the Puppetfile. The Puppetfile is used by Puppet Bolt to install the two Bolt modules that the provisioner depends upon, as mentioned above. Note that Puppetfile actually points to an unmerged pull request:

```ruby
# Modules from the Puppet Forge.
mod 'danieldreier/autosign'
mod 'puppetlabs/puppet_agent',
  :git => 'https://github.com/alexharv074/puppetlabs-puppet_agent.git',
  :ref => 'MODULES-9981-add_amazon_linux_2_support_to_install_task'
```

At the time of writing, there was no support for Amazon Linux 2 in the puppetlabs/puppet_agent `puppet_agent::install` task. I have added some support although foresee some delays in getting it merged. Hopefully that feature will be merged soon. If so, this file would be:

```ruby
mod 'danieldreier/autosign'
mod 'puppetlabs/puppet_agent'
```

### The Terraform code

#### main.tf

I have all my code in main.tf. The full contents of that file are:

```js
variable "key_name" {
  description = "The name of the EC2 key pair to use"
  default     = "default"
}

variable "key_file" {
  description = "The private key for the ec2-user used in SSH connections and by Puppet Bolt"
  default     = "~/.ssh/default.pem"
}

locals {
  linux_instance_type   = "t2.micro"
  windows_instance_type = "t2.large"
}

data "aws_ami" "amazon_linux_2" {
  owners      = ["amazon"]
  most_recent = true

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-ebs"]
  }
}

data "aws_ami" "windows_2012R2" {
  most_recent = "true"
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["Windows_Server-2012-R2_RTM-English-64Bit-Base-*"]
  }
}

data "template_file" "user_data" {
  template = file("${path.module}/user_data/master.sh")
}

data "template_file" "winrm" {
  template = file("${path.module}/user_data/win_agent.xml")
}

resource "aws_instance" "master" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = local.linux_instance_type
  key_name      = var.key_name
  user_data     = data.template_file.user_data.rendered

  provisioner "remote-exec" {
    on_failure = continue
    inline = [
      "sudo sh -c 'while ! grep -q Cloud-init.*finished /var/log/cloud-init-output.log; do sleep 20; done'"
    ]

    connection {
      host        = self.public_ip
      user        = "ec2-user"
      private_key = file(var.key_file)
    }
  }
}

resource "aws_instance" "linux_agent" {
  ami           = data.aws_ami.ami.id
  instance_type = local.linux_instance_type
  key_name      = var.key_name

  provisioner "puppet" {
    use_sudo    = true
    server      = aws_instance.master.public_dns
    server_user = "ec2-user"

    connection {
      host        = self.public_ip
      user        = "ec2-user"
      private_key = file(var.key_file)
    }
  }

  depends_on = [aws_instance.master]
}

resource "aws_instance" "win_agent" {
  ami               = data.aws_ami.windows_2012R2.image_id
  instance_type     = local.windows_instance_type
  key_name          = var.key_name
  get_password_data = true

  timeouts {
    create = "15m"
  }

  provisioner "puppet" {
    open_source = true
    server      = aws_instance.master.public_dns
    server_user = "ec2-user"

    connection {
      host     = self.public_ip
      type     = "winrm"
      user     = "Administrator"
      password = rsadecrypt(self.password_data, file(var.key_file))
    }
  }

  user_data  = data.template_file.winrm.rendered
  depends_on = [aws_instance.master]
}
```

#### About the Puppet Master

The Puppet provisioner assumes that you already have a Puppet Master runnings somewhere, and it is not the provisioner's job to help you build that. Also, building the Puppet Master threw some of the biggest challenges, so keep that in mind when reviewing the overall complexity of this solution.

##### user_data

To configure the Puppet Master, I wrote the following shell script that is called from user_data:

```bash
#!/usr/bin/env bash

# Without $HOME, a message is seen in cloud-init-output.log during autosign:
#   couldn't find login name -- expanding `~'
export HOME='/root'

install_puppetserver() {
  wget https://yum.puppet.com/puppet6-release-el-7.noarch.rpm
  rpm -Uvh puppet6-release-el-7.noarch.rpm
  yum-config-manager --enable puppet6
  yum -y install puppetserver
}

configure_puppetserver() {
  echo 'export PATH=/opt/puppetlabs/puppet/bin:$PATH' \
    >> /etc/profile.d/puppet-agent.sh
  . /etc/profile.d/puppet-agent.sh
  sed -i '
    s/JAVA_ARGS.*/JAVA_ARGS="-Xms512m -Xmx512m"/
    ' /etc/sysconfig/puppetserver # workaround for t2.micro's 1GB RAM.
  local public_hostname=$(curl \
    http://169.254.169.254/latest/meta-data/public-hostname)
  puppetserver ca setup \
    --subject-alt-names "$public_hostname",localhost,puppet
  echo "127.0.0.1  puppet" >> /etc/hosts
}

configure_autosign() {
  gem install autosign
  mkdir -p -m 750 /var/autosign
  chown puppet: /var/autosign
  touch /var/log/autosign.log
  chown puppet: /var/log/autosign.log
  autosign config setup
  sed -i '
    s!journalfile:.*!journalfile: "/var/autosign/autosign.journal"!
    ' /etc/autosign.conf
  puppet config set \
    --section master autosign /opt/puppetlabs/puppet/bin/autosign-validator
  systemctl restart puppetserver
}

deploy_code() {
  yum -y install git
  rm -rf /etc/puppetlabs/code/environments/production
  git clone \
    https://github.com/alexharv074/terraform-puppet-provisioner-test.git \
    /etc/puppetlabs/code/environments/production
}

main() {
  install_puppetserver
  configure_puppetserver
  configure_autosign
  deploy_code
}

main
```

Notice there is autosigning configuration provided by the autosign Ruby Gem. Your Puppet Master needs that configuration to support the Puppet Terraform provisioner. Also notice that IU had to get the Public Hostname from the EC2 instance Meta Data using curl. That is to workaround the fact that there seems to be no way to populate a Terraform template with the generated self.public_dns other than inside a connection or provisioner block!<sup>2</sup>

##### Remote exec provisioner

Also note the following "hack" to get Terraform to stop and wait before marking the Puppet Master's aws_instance state as "created". I refer to this code:

```js
  provisioner "remote-exec" {
    on_failure = continue
    inline = [
      "sudo sh -c 'while ! grep -q Cloud-init.*finished /var/log/cloud-init-output.log; do sleep 20; done'"
    ]
  }
```

The code uses a remote-exec provisioner to monitor the /var/log/cloud-init-output.log every 20 seconds for the a message that Cloud-init has finished. Is there a better way? Let me know! This is apparently the only way to do this because Terraform has no equivalent of CloudFormation's [cfn-signal](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-signal.html) to signal that a resource has been "created". See also the line in the agent configs:

```js
  depends_on = [aws_instance.master]
```

That's where I tell the agents to wait for the master to be created.

##### Default EC2 key pair

I have assumed that you have an EC2 key pair in your AWS account called "default". If you don't, you can create one using:

```text
▶ aws ec2 create-key-pair --key-name default
```

Or you could set a Terraform variable to point to another key you want to use. For example:

```text
▶ export TF_VAR_key_name='my_key'
```

#### The Amazon Linux 2 agent

The code for the Linux agent node is this:

```js
resource "aws_instance" "linux_agent" {
  ami           = data.aws_ami.ami.id
  instance_type = local.instance_type
  key_name      = var.key_name

  provisioner "puppet" {
    use_sudo    = true
    server      = aws_instance.master.public_dns
    server_user = "ec2-user"

    connection {
      host        = self.public_ip
      type        = "ssh" // This could be omitted after my above-mentioned patch is merged.
      user        = "ec2-user"
      private_key = file(var.key_file)
    }
  }

  depends_on = [aws_instance.master]
}
```

Things to note here:

- The connection block is used by the provisioner to connect to the Puppet Agent node.
- The settings `server` and `server_user` refer to the Puppet Master node. In my case, I have the Puppet Master managed in Terraform too although I can foresee others could have their Puppet Masters on long-lived pets etc.
- At the time of writing, the private key needed to connect to the Puppet Master lives in Puppet Bolt's configuration in the bolt.yaml file. I find this surprising and I'm going to raise a patch if I can to change this so that the private_key to connect to the Puppet Master will be specified in Terraform too.

#### Connection type SSH

I am one of those people who doesn't like to overspecify things in code and I tend to use default values where possible. I tried to do that for the SSH connection blocks for the Puppet Master and Agent aws_instances. I then ran into a quite confusing bug that led me on a goose chase through both the Terraform & Bolt code bases! That's why there's a comment there that points to [this](https://github.com/hashicorp/terraform/issues/23004) Terraform issue that I raised.

In the end I fixed that bug in this open pull request [here](https://github.com/hashicorp/terraform/pull/23057). At the time of writing, it is unmerged and will probably go in to Terraform 0.12.11. If you have a lower Terraform, just make sure you specify the connection type on Linux explicitly as "ssh".

#### The Windows 2012 node

And here is the Windows agent node code:

```js
resource "aws_instance" "win_agent" {
  ami               = data.aws_ami.windows_2012R2.image_id
  instance_type     = "t2.large"
  key_name          = var.key_name
  get_password_data = true

  timeouts {
    create = "15m"
  }

  provisioner "puppet" {
    open_source = true
    server      = aws_instance.master.public_dns
    server_user = "ec2-user"

    connection {
      host     = self.public_ip
      type     = "winrm"
      user     = "Administrator"
      password = rsadecrypt(self.password_data, file(var.key_file))
    }
  }

  user_data  = data.template_file.winrm.rendered
  depends_on = [aws_instance.master]
}
```

This is much the same as the Amazon Linux 2 configuration other than the password field that is passed in the connection block. There, I used the same EC2 user key to get the Administrator password, which is passed to the Puppet provisioner to be used by Bolt to connect to the Windows agent node and install the Puppet agent software.

### The Puppet code

Finally, I have the Puppet code itself inside manifests/site.pp:

```js
node default {
  notify { "Hello world from ${facts['hostname']}!": }
}
```

Note that this code is made available in the deploy_code function in the Puppet Master UserData above.

### Running it

First run the setup script.

```text
▶ bash -x setup.sh
```

Then run terraform apply:

```text
▶ terraform init
▶ terraform apply -auto-approve
```

### Expected output

```text
▶ terraform apply -auto-approve
data.template_file.winrm: Refreshing state...
data.template_file.user_data: Refreshing state...
data.aws_ami.ami: Refreshing state...
data.aws_ami.windows_2012R2: Refreshing state...
aws_instance.master: Creating...
aws_instance.master: Still creating... [10s elapsed]
aws_instance.master: Still creating... [20s elapsed]
aws_instance.master: Still creating... [30s elapsed]
aws_instance.master: Provisioning with 'remote-exec'...
aws_instance.master (remote-exec): Connecting to remote host via SSH...
aws_instance.master (remote-exec):   Host: 13.239.139.194
aws_instance.master (remote-exec):   User: ec2-user
aws_instance.master (remote-exec):   Password: false
aws_instance.master (remote-exec):   Private key: true
aws_instance.master (remote-exec):   Certificate: false
aws_instance.master (remote-exec):   SSH Agent: true
aws_instance.master (remote-exec):   Checking Host Key: false
aws_instance.master: Still creating... [40s elapsed]
aws_instance.master (remote-exec): Connecting to remote host via SSH...
aws_instance.master (remote-exec):   Host: 13.239.139.194
aws_instance.master (remote-exec):   User: ec2-user
aws_instance.master (remote-exec):   Password: false
aws_instance.master (remote-exec):   Private key: true
aws_instance.master (remote-exec):   Certificate: false
aws_instance.master (remote-exec):   SSH Agent: true
aws_instance.master (remote-exec):   Checking Host Key: false
aws_instance.master: Still creating... [50s elapsed]
aws_instance.master: Still creating... [1m0s elapsed]
aws_instance.master (remote-exec): Connecting to remote host via SSH...
aws_instance.master (remote-exec):   Host: 13.239.139.194
aws_instance.master (remote-exec):   User: ec2-user
aws_instance.master (remote-exec):   Password: false
aws_instance.master (remote-exec):   Private key: true
aws_instance.master (remote-exec):   Certificate: false
aws_instance.master (remote-exec):   SSH Agent: true
aws_instance.master (remote-exec):   Checking Host Key: false
aws_instance.master (remote-exec): Connecting to remote host via SSH...
aws_instance.master (remote-exec):   Host: 13.239.139.194
aws_instance.master (remote-exec):   User: ec2-user
aws_instance.master (remote-exec):   Password: false
aws_instance.master (remote-exec):   Private key: true
aws_instance.master (remote-exec):   Certificate: false
aws_instance.master (remote-exec):   SSH Agent: true
aws_instance.master (remote-exec):   Checking Host Key: false
aws_instance.master (remote-exec): Connected!
aws_instance.master: Still creating... [1m10s elapsed]
aws_instance.master: Still creating... [1m20s elapsed]
aws_instance.master: Still creating... [1m30s elapsed]
aws_instance.master: Still creating... [1m40s elapsed]
aws_instance.master: Still creating... [1m50s elapsed]
aws_instance.master: Still creating... [2m0s elapsed]
aws_instance.master: Still creating... [2m10s elapsed]
aws_instance.master: Still creating... [2m20s elapsed]
aws_instance.master: Still creating... [2m30s elapsed]
aws_instance.master: Still creating... [2m40s elapsed]
aws_instance.master: Still creating... [2m50s elapsed]
aws_instance.master: Still creating... [3m0s elapsed]
aws_instance.master: Still creating... [3m10s elapsed]
aws_instance.master: Creation complete after 3m17s [id=i-0d126b0f634539c45]
aws_instance.linux_agent: Creating...
aws_instance.win_agent: Creating...
aws_instance.win_agent: Still creating... [10s elapsed]
aws_instance.linux_agent: Still creating... [10s elapsed]
aws_instance.linux_agent: Still creating... [20s elapsed]
aws_instance.win_agent: Still creating... [20s elapsed]
aws_instance.linux_agent: Provisioning with 'puppet'...
aws_instance.linux_agent (puppet): Connecting to remote host via SSH...
aws_instance.linux_agent (puppet):   Host: 54.252.134.38
aws_instance.linux_agent (puppet):   User: ec2-user
aws_instance.linux_agent (puppet):   Password: false
aws_instance.linux_agent (puppet):   Private key: true
aws_instance.linux_agent (puppet):   Certificate: false
aws_instance.linux_agent (puppet):   SSH Agent: true
aws_instance.linux_agent (puppet):   Checking Host Key: false
aws_instance.win_agent: Still creating... [30s elapsed]
aws_instance.linux_agent: Still creating... [30s elapsed]
aws_instance.linux_agent (puppet): Connecting to remote host via SSH...
aws_instance.linux_agent (puppet):   Host: 54.252.134.38
aws_instance.linux_agent (puppet):   User: ec2-user
aws_instance.linux_agent (puppet):   Password: false
aws_instance.linux_agent (puppet):   Private key: true
aws_instance.linux_agent (puppet):   Certificate: false
aws_instance.linux_agent (puppet):   SSH Agent: true
aws_instance.linux_agent (puppet):   Checking Host Key: false
aws_instance.win_agent: Still creating... [40s elapsed]
aws_instance.linux_agent: Still creating... [40s elapsed]
aws_instance.linux_agent (puppet): Connecting to remote host via SSH...
aws_instance.linux_agent (puppet):   Host: 54.252.134.38
aws_instance.linux_agent (puppet):   User: ec2-user
aws_instance.linux_agent (puppet):   Password: false
aws_instance.linux_agent (puppet):   Private key: true
aws_instance.linux_agent (puppet):   Certificate: false
aws_instance.linux_agent (puppet):   SSH Agent: true
aws_instance.linux_agent (puppet):   Checking Host Key: false
aws_instance.linux_agent (puppet): Connecting to remote host via SSH...
aws_instance.linux_agent (puppet):   Host: 54.252.134.38
aws_instance.linux_agent (puppet):   User: ec2-user
aws_instance.linux_agent (puppet):   Password: false
aws_instance.linux_agent (puppet):   Private key: true
aws_instance.linux_agent (puppet):   Certificate: false
aws_instance.linux_agent (puppet):   SSH Agent: true
aws_instance.linux_agent (puppet):   Checking Host Key: false
aws_instance.linux_agent (puppet): Connected!
aws_instance.linux_agent (puppet): ip-172-31-10-49.ap-southeast-2.compute.internal
aws_instance.linux_agent: Still creating... [50s elapsed]
aws_instance.win_agent: Still creating... [50s elapsed]
aws_instance.linux_agent: Still creating... [1m0s elapsed]
aws_instance.win_agent: Still creating... [1m0s elapsed]
aws_instance.win_agent: Still creating... [1m10s elapsed]
aws_instance.linux_agent: Still creating... [1m10s elapsed]
aws_instance.win_agent: Still creating... [1m20s elapsed]
aws_instance.linux_agent: Still creating... [1m20s elapsed]
aws_instance.win_agent: Provisioning with 'puppet'...
aws_instance.win_agent (puppet): Connecting to remote host via WinRM...
aws_instance.win_agent (puppet):   Host: 13.211.55.90
aws_instance.win_agent (puppet):   Port: 5985
aws_instance.win_agent (puppet):   User: Administrator
aws_instance.win_agent (puppet):   Password: true
aws_instance.win_agent (puppet):   HTTPS: false
aws_instance.win_agent (puppet):   Insecure: false
aws_instance.win_agent (puppet):   NTLM: false
aws_instance.win_agent (puppet):   CACert: false
aws_instance.win_agent (puppet): Connected!
aws_instance.win_agent (puppet): WIN-IPE5577KSBA
aws_instance.linux_agent (puppet): Info: Downloaded certificate for ca from ec2-13-239-139-194.ap-southeast-2.compute.amazonaws.com
aws_instance.linux_agent (puppet): Info: Downloaded certificate revocation list for ca from ec2-13-239-139-194.ap-southeast-2.compute.amazonaws.com
aws_instance.linux_agent (puppet): Info: Creating a new RSA SSL key for ip-172-31-10-49.ap-southeast-2.compute.internal
aws_instance.win_agent (puppet): ap-southeast-2.compute.internal
aws_instance.linux_agent (puppet): Info: csr_attributes file loading from /etc/puppetlabs/puppet/csr_attributes.yaml
aws_instance.linux_agent (puppet): Info: Creating a new SSL certificate request for ip-172-31-10-49.ap-southeast-2.compute.internal
aws_instance.linux_agent (puppet): Info: Certificate Request fingerprint (SHA256): E3:E8:AD:42:EC:76:EE:F0:DF:47:F9:D1:65:6B:8C:46:0B:59:B2:1A:26:5B:56:B7:55:87:1C:B9:7E:E6:BA:3E
aws_instance.linux_agent (puppet): Info: Downloaded certificate for ip-172-31-10-49.ap-southeast-2.compute.internal from ec2-13-239-139-194.ap-southeast-2.compute.amazonaws.com
aws_instance.win_agent: Still creating... [1m30s elapsed]
aws_instance.linux_agent: Still creating... [1m30s elapsed]
aws_instance.linux_agent (puppet): Info: Using configured environment 'production'
aws_instance.linux_agent (puppet): Info: Retrieving pluginfacts
aws_instance.linux_agent (puppet): Info: Retrieving plugin
aws_instance.linux_agent (puppet): Info: Retrieving locales


aws_instance.win_agent (puppet):     Directory: C:\ProgramData\PuppetLabs\Puppet


aws_instance.win_agent (puppet): Mode                LastWriteTime     Length Name
aws_instance.win_agent (puppet): ----                -------------     ------ ----
aws_instance.win_agent (puppet): d----        10/12/2019  11:47 AM            etc


aws_instance.linux_agent (puppet): Info: Caching catalog for ip-172-31-10-49.ap-southeast-2.compute.internal
aws_instance.linux_agent (puppet): Info: Applying configuration version '1570880860'
aws_instance.linux_agent (puppet): Notice: Hello world from ip-172-31-10-49!
aws_instance.linux_agent (puppet): Notice: /Stage[main]/Main/Node[default]/Notify[Hello world from ip-172-31-10-49!]/message: defined 'message' as 'Hello world from ip-172-31-10-49!'
aws_instance.linux_agent (puppet): Info: Creating state file /opt/puppetlabs/puppet/cache/state/state.yaml
aws_instance.linux_agent (puppet): Notice: Applied catalog in 0.01 seconds
aws_instance.linux_agent: Creation complete after 1m33s [id=i-06b88138c2feda4cf]
aws_instance.win_agent: Still creating... [1m40s elapsed]
aws_instance.win_agent: Still creating... [1m50s elapsed]
aws_instance.win_agent: Still creating... [2m0s elapsed]
aws_instance.win_agent: Still creating... [2m10s elapsed]
aws_instance.win_agent: Still creating... [2m20s elapsed]
aws_instance.win_agent: Still creating... [2m30s elapsed]
aws_instance.win_agent: Still creating... [2m40s elapsed]
aws_instance.win_agent (puppet): Info: Downloaded certificate for ca from ec2-13-239-139-194.ap-southeast-2.compute.amazonaws.com
aws_instance.win_agent (puppet): Info: Downloaded certificate revocation list for ca from ec2-13-239-139-194.ap-southeast-2.compute.amazonaws.com
aws_instance.win_agent (puppet): Info: Creating a new RSA SSL key for win-ipe5577ksba.ap-southeast-2.compute.internal
aws_instance.win_agent: Still creating... [2m50s elapsed]
aws_instance.win_agent (puppet): Info: csr_attributes file loading from C:/ProgramData/PuppetLabs/puppet/etc/csr_attributes.yaml
aws_instance.win_agent (puppet): Info: Creating a new SSL certificate request for win-ipe5577ksba.ap-southeast-2.compute.internal
aws_instance.win_agent (puppet): Info: Certificate Request fingerprint (SHA256): A1:C0:D3:AD:24:C7:80:67:F1:F4:97:FC:06:E2:16:01:12:DA:02:5F:AA:2F:57:98:9F:7D:2A:34:42:3C:D3:50
aws_instance.win_agent (puppet): Info: Downloaded certificate for win-ipe5577ksba.ap-southeast-2.compute.internal from ec2-13-239-139-194.ap-southeast-2.compute.amazonaws.com
aws_instance.win_agent (puppet): Info: Using configured environment 'production'
aws_instance.win_agent (puppet): Info: Retrieving pluginfacts
aws_instance.win_agent (puppet): Info: Retrieving plugin
aws_instance.win_agent (puppet): Info: Retrieving locales
aws_instance.win_agent (puppet): Info: Caching catalog for win-ipe5577ksba.ap-southeast-2.compute.internal
aws_instance.win_agent (puppet): Info: Applying configuration version '1570880943'
aws_instance.win_agent (puppet): Notice: Hello world from WIN-IPE5577KSBA!
aws_instance.win_agent (puppet): Notice: /Stage[main]/Main/Node[default]/Notify[Hello world from WIN-IPE5577KSBA!]/message: defined 'message' as 'Hello world from WIN-IPE5577KSBA!'
aws_instance.win_agent (puppet): Info: Creating state file C:/ProgramData/PuppetLabs/puppet/cache/state/state.yaml
aws_instance.win_agent (puppet): Notice: Applied catalog in 0.02 seconds
aws_instance.win_agent: Creation complete after 2m55s [id=i-07da31c6a0bf6ce14]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
```

## Discussion

It has been a journey to set all of this up that, as mentioned already, led me to submit patches into both Terraform and one of the Bolt modules. I do hope it is useful to get others up to speed quickly.

For those who are already using Terraform, Puppet Bolt, and Puppet to manage their EC2 infrastructure, I expect that this provisioner will be useful and they probably will want to think about using it. When it is all set up it feels quite clean and it is a nice user experience. Remember, as I mentioned already, that most of the complexity in the solution relates to setting up the Puppet Master in Terraform. That will not be a consideration for most people who are already using Puppet Masters.

The more interesting question might be this: Who should use this if they are _not_ already using Terraform, Puppet Bolt, and Puppet to manage their EC2 instances? How _should_ you manage a fleet of EC2 instances using Terraform?

HashiCorp [say](https://www.terraform.io/docs/provisioners/index.html) that provisioners - any provisioners, whether Chef, Puppet, local-exec etc - should be used as a "last resort" and of the configuration management provisioners specifically:

> As a convenience to users who are forced to use generic operating system distribution images, Terraform includes a number of specialized provisioners for launching specific configuration management products.
>
> We strongly recommend not using these, and instead running system configuration steps during a custom image build process. For example, HashiCorp Packer offers a similar complement of configuration management provisioners and can run their installation steps during a separate build process, before creating a system disk image that you can deploy many times.
>
> If you are using configuration management software that has a centralized server component, you will need to delay the registration step until the final system is booted from your custom image.

This is a statement of HashiCorp's strongly opinionated view of how to do configuration management of course and it is debatable. Equally, it is possible to do configuration management using only Terraform's features and UserData scripts. This also works fine ... most of the time!

But if baking machine images isn't the best fit for your problem and you foresee yourself also outgrowing UserData - and believe me you need to think hard about this because you don't want to ever run into the limits of UserData as your configuration management solution because you'll probably discover that you have no choice other than to rewrite everything! - that is, if your use-case may grow to include configuration management of complex applications running on Linux or Windows - then the use of a provisioner like the Puppet provisioner (or the Chef provisioner or some of the others) deserves consideration.

I can imagine that the requirement to also have Puppet Bolt on the machine running Terraform is going to be an issue for some users. If this provisioner is the _only_ reason to use Puppet Bolt, you may decide to do your configuration management another way. But with that said, Puppet Bolt is a quite powerful tool that also deserves consideration.

## See also

- Martez Reed, 10th July 2019, [Terraform Puppet Provisioner](https://www.greenreedtech.com/terraform-puppet-provisioner/).

- Tim Sharpe's (the provisioner author's) [test code](https://github.com/rodjek/terraform-puppet-example).

---

<sup>1</sup> Note that I refer throughout to the "Puppet Master" using the traditional terminology, although I probably should say "Puppet Server".<br>
<sup>2</sup> See also Tim Sharpe's apparent solution to the same problem [here](https://github.com/rodjek/terraform-puppet-example/blob/master/example.tf#L33-L36).
