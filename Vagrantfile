# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"

  config.vm.provider "virtualbox" do |v|
    v.memory = 256
    v.cpus = 1
  end

  config.vm.define "ns01" do |ns01|
    ns01.vm.network "private_network", ip: "192.168.50.10", virtualbox__intnet: "net1"
    ns01.vm.hostname = "ns01"
    ns01.vm.provision "shell", inline: <<-SHELL
      mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh
    SHELL
    ns01.vm.provision "shell", path: "ns01.sh"
  end

  config.vm.define "client" do |client|
    client.vm.network "private_network", ip: "192.168.50.15", virtualbox__intnet: "net1"
    client.vm.hostname = "client"
    client.vm.provision "shell", path: "client.sh"
  end

end
