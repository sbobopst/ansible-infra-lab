# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

# Lab VM definitions
MACHINES = {
  "webserver01" => { ip: "192.168.56.10", memory: 2048, cpus: 4 },
  "dbserver01"  => { ip: "192.168.56.11", memory: 2048, cpus: 4 },
  "monitor01"   => { ip: "192.168.56.12", memory: 2048, cpus: 4 },
}

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.box = "generic/ubuntu2204"

  # Copy id_ansible public key into every VM so Ansible can connect
  config.vm.provision "shell" do |s|
    ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_ansible.pub").first.strip
    s.inline = <<-SHELL
      echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys
      chmod 600 /home/vagrant/.ssh/authorized_keys
    SHELL
  end

  MACHINES.each do |name, opts|
    config.vm.define name do |node|
      node.vm.hostname = name
      node.vm.network "private_network", ip: opts[:ip]

      node.vm.provider :libvirt do |lv|
        lv.memory = opts[:memory]
        lv.cpus   = opts[:cpus]
        lv.driver = "kvm"
      end
    end
  end

end