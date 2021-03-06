# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'

# get details of boxes to build
boxes = YAML.load_file('./hack/vagrant/boxes.yaml')

# API version
VAGRANTFILE_API_VERSION = 2
DEFAULT_BASE_BOX = 'centos/7'

def serverIP(num)
  return "172.17.100.#{num+50}"
end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # Populate boxes hosts file
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.manage_guest = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true

  boxes.each do |boxes|
    NUMBER = 1
    config.vm.define boxes['name'] do |srv|
      # OS and hostname
      srv.vm.box = boxes['box'] ||= DEFAULT_BASE_BOX
      srv.vm.box_version = boxes['box_version'] if boxes.key? 'box_version'
      srv.vm.box_url = boxes['box_url'] if boxes.key? 'box_url'
      srv.vm.hostname = boxes['hostname']
      srv.hostmanager.aliases = ["#{boxes['hostname']}.localdomain", boxes['hostname']]
      
      # Networking.  By default a NAT interface is added.
      # Add an internal network like this:
      #   srv.vm.network 'private_network', type: 'dhcp', virtualbox__intnet: true
      # Add a bridged network
      if boxes['public_network']
        if boxes['public_network']['ip']
          srv.vm.network 'public_network', bridge: boxes['public_network']['bridge'], ip: boxes['public_network']['ip']
        else
          srv.vm.network 'public_network', bridge: boxes['public_network']['bridge']
        end
      end

      # Set private network insterface
      srv.vm.network :private_network, ip: serverIP(NUMBER)

      # Copy software packages to tmp
      if boxes['forward_port']
        boxes['forward_port'].each do |forwarded|
          srv.vm.network :forwarded_port, guest: forwarded['port'], host: forwarded['expose'], host_ip: '127.0.0.1', id: forwarded['name']
        end
      end

      # VirtualBox
      srv.vm.provider 'virtualbox' do |vb|
        vb.customize ['modifyvm', :id, '--cpus', boxes['cpus']] if boxes.key? 'cpus'
        vb.customize ['modifyvm', :id, '--cpuexecutioncap', boxes['cpu_execution_cap']] if boxes.key? 'cpuexecutioncap'
        vb.customize ['modifyvm', :id, '--memory', boxes['memory']] if boxes.key? 'memory'
        vb.customize ['modifyvm', :id, '--name', boxes['name']] if boxes.key? 'name'
        vb.customize ['modifyvm', :id, '--description', boxes['description']] if boxes.key? 'description'
        vb.customize ['modifyvm', :id, '--paravirtprovider', boxes['paravirtprovider']] if boxes.key? 'paravirtprovider'
      end

      # Hyper-V
      srv.vm.provider 'hyperv' do |hv|
        hv.cpus   = boxes['cpus'] if boxes.key? 'cpus'
        hv.memory = boxes['ram'] if boxes.key? 'memory'
        hv.vmname = boxes['name'] if boxes.key? 'name'
      end

      # Copy cloud-init files to tmp and provision
      if boxes['provision']
        srv.vm.provision :file, :source => boxes['provision']['meta-data'], :destination => '/tmp/vagrant/cloud-init/nocloud-net/meta-data'
        srv.vm.provision :file, :source => boxes['provision']['user-data'], :destination => '/tmp/vagrant/cloud-init/nocloud-net/user-data'
        srv.vm.provision :shell, :path => boxes['provision']['cloud-init'], :args => boxes['name']
      end
      
      # Run Ansible playbook
      if boxes['ansible']
        # Shared folders
        srv.vm.synced_folder './',
        '/vagrant',
        disabled: false,
        type: "rsync",
        rsync__auto: true,
        rsync__exclude: ['.vagrant/', '.vscode/', '.git/']
        
        srv.vm.provision "ansible_local" do |ansible|
          ansible.verbose="vvv"
          ansible.become = true
          ansible.verbose = true
          ansible.playbook = boxes['ansible']['playbook']
          ansible.galaxy_roles_path = '/vagrant/roles'
          ansible.galaxy_role_file = "requirements.yaml"
          # ansible.install_mode = "pip3_args_only"
          # ansible.pip_args = "-r requirements.txt"
          # ansible.extra_vars = {
          #   ansible_python_interpreter: "/usr/bin/env python3",
          #   ansible_stdout_callback: "debug"
          # }
        end
      
      else
      
        # Copy software packages to tmp
        if boxes['packages']
          boxes['packages'].each do |package|
            srv.vm.provision :file, :source => package, :destination => '/tmp/'
          end
        end

        # Copy configuration files to tmp
        if boxes['files']
          boxes['files'].each do |file|
            srv.vm.provision :file, :source => file, :destination => '/tmp/'
          end
        end

        # Copy script
        if boxes['scripts']
          boxes['scripts'].each do |script|
            srv.vm.provision :file, :source => script, :destination => '/tmp/'
          end
        end

        # Run installation script
        if boxes['install']
          srv.vm.provision :shell, :path => boxes['install']
        end
      end
      
      NUMBER += 1
    end
  end
end