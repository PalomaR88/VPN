# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.define :vpnserv do |vpnserv|
    vpnserv.vm.box = "debian/buster64"
    vpnserv.vm.hostname = "servidor"
    vpnserv.vm.network :public_network,:bridge=>"wlp1s0"
    vpnserv.vm.network :private_network, ip: "192.168.100.1", virtualbox__intnet:"mired1"
  end
  config.vm.define :vpnsts do |vpnsts|
    vpnsts.vm.box = "debian/buster64"
    vpnsts.vm.hostname = "cliente"
    vpnsts.vm.network :private_network, ip: "192.168.100.2", virtualbox__intnet:"mired1"
  end
end

