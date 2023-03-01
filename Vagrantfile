# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
$mach_quant = 2
Vagrant.configure("2") do |config|
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = "1"
  end
  config.vm.box = "ubuntu/bionic64"
  config.vm.boot_timeout = 600
  

  (1..$mach_quant).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.network "private_network", ip: "192.168.1.#{1+i}"
      node.vm.hostname = "node#{i}"
   end
 end
end