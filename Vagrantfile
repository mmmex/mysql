# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # config.vm.synced_folder ".", "/vagrant", mount_options: ["dmode=700,fmode=600"]
  # config.vm.synced_folder "./config", "/vagrant/config", mount_options: ["dmode=755,fmode=755"]
  
  # run slave first
  config.vm.define "mysqlslave" do |mysqlslave|
    mysqlslave.vm.box = "bento/ubuntu-20.04"
    mysqlslave.vm.hostname = 'mysqlslave'
    # mysqlslave.vm.synced_folder "./data/slave", "/var/lib/mysql_vagrant" , id: "mysql",
    # owner: 108, group: 113,  # owner: "mysql", group: "mysql",
    # mount_options: ["dmode=775,fmode=664"]

    mysqlslave.vm.network :private_network, ip: "192.168.100.12"

    mysqlslave.vm.provider :virtualbox do |v|
      v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      v.customize ["modifyvm", :id, "--memory", 512]
      v.customize ["modifyvm", :id, "--name", "mysqlslave"]
    end

    # mysqlslave.vm.provision :shell, path: "bootstrap-slave.sh"
  end

  config.vm.define "mysqlmaster" do |mysqlmaster|
    mysqlmaster.vm.box = "bento/ubuntu-20.04"
    mysqlmaster.vm.hostname = 'mysqlmaster'
    # mysqlmaster.vm.synced_folder "./data/master", "/var/lib/mysql_vagrant" , id: "mysql",
    # owner: 108, group: 113,  # owner: "mysql", group: "mysql",
    # mount_options: ["dmode=775,fmode=664"]

    mysqlmaster.vm.network :private_network, ip: "192.168.100.11"

    mysqlmaster.vm.provider :virtualbox do |v|
      v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      v.customize ["modifyvm", :id, "--memory", 512]
      v.customize ["modifyvm", :id, "--name", "mysqlmaster"]      
    end

    # mysqlmaster.vm.provision :shell, path: "bootstrap-master.sh"
    mysqlmaster.vm.provision "ansible" do |ansible|
      # ansible.compatibility_mode = "2.0"
      ansible.become = true
      ansible.limit = "all"
      # ansible.inventory_path = "ansible/inventory/hosts"
      # ansible.config_file = "ansible/ansible.cfg"
      # ansible.verbose = "vvvv"
      ansible.playbook = "ansible/provision.yml"
    end 
  end
   
end
