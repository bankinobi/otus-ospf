# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :ospf1 => {
    :box_name => "centos/7",
    :net => [
                {ip: '192.168.10.1', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "area0"},
                {ip: '192.168.30.2', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "area0"},
            ]
  },
  :ospf2 => {
    :box_name => "centos/7",
    :net => [
                {ip: '192.168.10.2', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "area0"},
                {ip: '192.168.20.1', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "area0"},
            ]
  },
  :ospf3 => {
    :box_name => "centos/7",
    :net => [
                {ip: '192.168.20.2', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "area0"},
                {ip: '192.168.30.1', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "area0"},
            ]
  }
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|

        box.vm.box = boxconfig[:box_name]
        box.vm.host_name = boxname.to_s

        boxconfig[:net].each do |ipconf|
          box.vm.network "private_network", ipconf
        end

        box.vm.provision "shell", inline: <<-SHELL
          mkdir -p ~root/.ssh
                cp ~vagrant/.ssh/auth* ~root/.ssh
        SHELL

        box.vm.provision "ansible" do |ansible|
          ansible.become = true
          ansible.verbose = "v"
          ansible.playbook = "provision/provision.yaml"
        end 

      end

  end

end
