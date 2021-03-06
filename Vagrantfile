# -*- mode: ruby -*-
# vi: set ft=ruby :

BOX_NAME = ENV['BOX_NAME'] || "ubuntu-raring"
BOX_URI = ENV['BOX_URI'] || "http://cloud-images.ubuntu.com/vagrant/raring/current/raring-server-cloudimg-amd64-vagrant-disk1.box"
VF_BOX_URI = ENV['BOX_URI'] || "http://nitron-vagrant.s3-website-us-east-1.amazonaws.com/vagrant_ubuntu_12.04.3_amd64_vmware.box"
AWS_BOX_URI = ENV['BOX_URI'] || "https://github.com/mitchellh/vagrant-aws/raw/master/dummy.box"
AWS_REGION = ENV['AWS_REGION'] || "us-east-1"
AWS_AMI = ENV['AWS_AMI'] || "ami-69f5a900"
AWS_INSTANCE_TYPE = ENV['AWS_INSTANCE_TYPE'] || 't1.micro'

FORWARD_DOCKER_PORTS = ENV['FORWARD_DOCKER_PORTS']

SSH_PRIVKEY_PATH = ENV["SSH_PRIVKEY_PATH"]
DOCKER_CPU_AMOUNT = ENV["DOCKER_CPU_AMOUNT"]

# Determine the Host OS to figure out where the Guest Additions are locally.
require 'rbconfig'
include RbConfig

case CONFIG['host_os']
  when /mswin|windows/i
    HOST_OS = "Windows"
  when /linux|arch/i
    HOST_OS = "Linux"
  when /darwin/i
    HOST_OS = "MacOS"
end
case HOST_OS
  when "Windows"
    GUEST_ADDITIONS_PATH = "C:\\Program files\\Oracle\\VirtualBox\\VBoxGuestAdditions.iso"
  when "Linux"
    GUEST_ADDITIONS_PATH = "/usr/share/virtualbox/VBoxGuestAdditions.iso"
  when "MacOS"
    GUEST_ADDITIONS_PATH = "/Applications/VirtualBox.app/Contents/MacOS/VBoxGuestAdditions.iso"
end

# A script to install docker.
$script = <<SCRIPT
# The username to add to the docker group will be passed as the first argument
# to the script.  If nothing is passed, default to "vagrant".
user="$1"
if [ -z "$user" ]; then
    user=vagrant
fi

# Adding an apt gpg key is idempotent.
wget -q -O - https://get.docker.io/gpg | apt-key add -

# Creating the docker.list file is idempotent, but it may overwrite desired
# settings if it already exists.  This could be solved with md5sum but it
# doesn't seem worth it.
echo 'deb http://get.docker.io/ubuntu docker main' > \
    /etc/apt/sources.list.d/docker.list

# Update remote package metadata.  'apt-get update' is idempotent.
apt-get update -q

# Install docker.  'apt-get install' is idempotent.
apt-get install -q -y lxc-docker

usermod -a -G docker "$user"

SCRIPT

# We need to install the virtualbox guest additions *before* we do the normal
# docker installation.  As such this script is prepended to the common docker
# install script above.  This allows the install of the backport kernel to
# trigger dkms to build the virtualbox guest module install.
$vbox_script = <<VBOX_SCRIPT + $script
# Install the VirtualBox guest additions if they aren't already installed.
  mount /dev/dvd /mnt || {
    echo 'Downloading VBox Guest Additions...'
    wget -cq http://dlc.sun.com.edgesuite.net/virtualbox/4.3.2/VBoxGuestAdditions_4.3.2.iso

    mount -o loop,ro /home/vagrant/VBoxGuestAdditions_4.3.2.iso /mnt
  }
  
  /mnt/VBoxLinuxAdditions.run install --force 
  umount /mnt
  rm -f /home/vagrant/*.iso
VBOX_SCRIPT

Vagrant::Config.run do |config|
  # Setup virtual machine box. This VM configuration code is always executed.
  config.vm.box = BOX_NAME
  config.vm.box_url = BOX_URI

  # Use the specified private key path if it is specified and not empty.
  if SSH_PRIVKEY_PATH
      config.ssh.private_key_path = SSH_PRIVKEY_PATH
  end

  config.ssh.forward_agent = true
end

# Providers were added on Vagrant >= 1.1.0
#
# NOTE: The vagrant "vm.provision" appends its arguments to a list and executes
# them in order.  If you invoke "vm.provision :shell, :inline => $script"
# twice then vagrant will run the script two times.  Unfortunately when you use
# providers and the override argument to set up provisioners (like the vbox
# guest extensions) they 1) don't replace the other provisioners (they append
# to the end of the list) and 2) you can't control the order the provisioners
# are executed (you can only append to the list).  If you want the virtualbox
# only script to run before the other script, you have to jump through a lot of
# hoops.
#
# Here is my only repeatable solution: make one script that is common ($script)
# and another script that is the virtual box guest *prepended* to the common
# script.  Only ever use "vm.provision" *one time* per provider.  That means
# every single provider has an override, and every single one configures
# "vm.provision".  Much saddness, but such is life.
Vagrant::VERSION >= "1.1.0" and Vagrant.configure("2") do |config|
  config.vm.provider :aws do |aws, override|
    username = "ubuntu"
    override.vm.box_url = AWS_BOX_URI
    override.vm.provision :shell, :inline => $script, :args => username
    aws.access_key_id = ENV["AWS_ACCESS_KEY"]
    aws.secret_access_key = ENV["AWS_SECRET_KEY"]
    aws.keypair_name = ENV["AWS_KEYPAIR_NAME"]
    override.ssh.username = username
    aws.region = AWS_REGION
    aws.ami    = AWS_AMI
    aws.instance_type = AWS_INSTANCE_TYPE
  end

  config.vm.provider :rackspace do |rs, override|
    override.vm.provision :shell, :inline => $script
    rs.username = ENV["RS_USERNAME"]
    rs.api_key  = ENV["RS_API_KEY"]
    rs.public_key_path = ENV["RS_PUBLIC_KEY"]
    rs.flavor   = /512MB/
    rs.image    = /Ubuntu/
  end

  config.vm.provider :vmware_fusion do |f, override|
    override.vm.box_url = VF_BOX_URI
    override.vm.synced_folder ".", "/vagrant", disabled: true
    override.vm.provision :shell, :inline => $script
    f.vmx["displayName"] = "docker"
  end

  config.vm.provider :virtualbox do |vb, override|
    override.vm.provision :shell, :inline => $vbox_script
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
    if !DOCKER_CPU_AMOUNT.nil?
      vb.customize ["modifyvm", :id, "--cpus", DOCKER_CPU_AMOUNT]
    end
    # Mount the local guest additions if they are available, so we have the proper version.
    if !GUEST_ADDITIONS_PATH.nil?
      vb.customize ["storageattach", :id, "--storagectl", "SATAController", "--port", "1", "--type", "dvddrive", "--medium", GUEST_ADDITIONS_PATH]
    end
  end
end

# If this is a version 1 config, virtualbox is the only option.  A version 2
# config would have already been set in the above provider section.
Vagrant::VERSION < "1.1.0" and Vagrant::Config.run do |config|
  config.vm.provision :shell, :inline => $vbox_script
end


if !FORWARD_DOCKER_PORTS.nil?
  Vagrant::VERSION < "1.1.0" and Vagrant::Config.run do |config|
    (49000..49900).each do |port|
      config.vm.forward_port port, port
    end
  end

  Vagrant::VERSION >= "1.1.0" and Vagrant.configure("2") do |config|
    (49000..49900).each do |port|
      config.vm.network :forwarded_port, :host => port, :guest => port, auto_correct: true
    end
  end
end
