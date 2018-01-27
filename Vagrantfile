# One Vagrantfile to rule them all!
#
# This is a generic Vagrantfile that can be used without modification in
# a variety of situations. Hosts and their properties are specified in
# `vagrant-hosts.yml`.
#
# See https://github.com/bertvv/ansible-skeleton/ for details
require 'rbconfig'
require 'yaml'
#TODO old require 'pp' 
require "time"

# set default LC_ALL for all BOXES
ENV["LC_ALL"] = "en_US.UTF-8"

# Set your default base box here
DEFAULT_BASE_BOX = 'bento/centos-7.4'

VAGRANTFILE_API_VERSION = '2'
VAGRANT_VERSION = "2.0.1"
PROJECT_NAME = '/' + File.basename(Dir.getwd)

# When set to `true`, Ansible will be forced to be run locally on the VM
# instead of from the host machine (provided Ansible is installed).
FORCE_LOCAL_RUN = false

# set custom vagrant-hosts file
vagranthosts = ENV['VAGRANTS_HOST'] ? ENV['VAGRANTS_HOST'] : 'vagrant-hosts.yml'
hosts = YAML.load_file(File.join(__dir__, vagranthosts))

# {{{ Helper functions

Vagrant.require_version ">= #{VAGRANT_VERSION}"

def provision_ansible(config, host)
  if run_locally?
    # Provisioning configuration for shell script.
    config.vm.provision 'shell' do |sh|
      sh.path = 'scripts/run-playbook-locally.sh'
    end
  else
    # Provisioning configuration for Ansible (for Mac/Linux hosts).
    config.vm.provision 'ansible' do |ansible|
      ansible.playbook = host.key?('playbook') ?
          "ansible/#{host['playbook']}" :
          "ansible/site.yml"
      ansible.become = true
    end
  end
end

def run_locally?
  windows_host? || FORCE_LOCAL_RUN
end

def windows_host?
  Vagrant::Util::Platform.windows?
end

# Set options for the network interface configuration. All values are
# optional, and can include:
# - ip (default = DHCP)
# - netmask (default value = 255.255.255.0
# - mac
# - auto_config (if false, Vagrant will not configure this network interface
# - intnet (if true, an internal network adapter will be created instead of a
#   host-only adapter)
def network_options(host)
  options = {}

  if host.key?('ip')
    options[:ip] = host['ip']
    options[:netmask] = host['netmask'] ||= '255.255.255.0'
  else
    options[:type] = 'dhcp'
  end

  options[:mac] = host['mac'].gsub(/[-:]/, '') if host.key?('mac')
  options[:auto_config] = host['auto_config'] if host.key?('auto_config')
  options[:virtualbox__intnet] = true if host.key?('intnet') && host['intnet']
  options
end

def custom_synced_folders(vm, host)
  return unless host.key?('synced_folders')
  folders = host['synced_folders']

  folders.each do |folder|
    vm.synced_folder folder['src'], folder['dest'], folder['options']
  end
end

# }}}


# Set options for shell provisioners to be run always. If you choose to include
# it you have to add a cmd variable with the command as data.
# 
# Use case: start symfony dev-server
#
# example: 
# shell_always:
#   - cmd: php /srv/google-dev/bin/console server:start 192.168.52.25:8080 --force
def shell_provisioners_always(vm, host)
  if host.has_key?('shell_always')
    scripts = host['shell_always']

    scripts.each do |script|
      vm.provision "shell", inline: script['cmd'], run: "always"
    end
  end
end

# }}}


# Adds forwarded ports to your vagrant machine so they are available from your phone
#
# example: 
#  forwarded_ports:
#    - guest: 88
#      host: 8080
def forwarded_ports(vm, host)
  if host.has_key?('forwarded_ports')
    ports = host['forwarded_ports']

    ports.each do |port|
      vm.network "forwarded_port", guest: port['guest'], host: port['host']
    end
  end
end

def tmp_set_keys_to_known_host(node, host)

  #$stdout.print "node => #{ node.inspect}\n"
  $stdout.print "host => #{host}\n"
  $stdout.print "host => #{host['name']}\n"
  id = %x(cat .vagrant/machines/#{host['name']}/virtualbox/id).chomp
  $stdout.print "id => #{id}\n"


  #$stdout.print "node.vm inspect => #{node.vm.inspect}\n"
  #$stdout.print "node.id=> #{node.vm.id}\n"
  #$stdout.print "node.vm.networks => #{node.vm.networks}\n"  
  #$stdout.print "node.vm.name => #{node.vm.name}\n"
  #get name of box
  #$stdout.print "node.vm.box => #{node.vm.box}\n"
  # $stdout.print "node.vm.box_version => #{node.vm.box_version.inspect}\n"

  #$stdout.print "node => #{node.index_uuid}\n"

  
end 

# copy file
def self.copy_file(src, dest)
  # w - Create an empty file for writing.
  File.open(dest, 'w') { |f| f.write(File.read(src)) }
end

def remove_added_kezs_by_vagrant_project()

  projecthome=Dir.getwd
  $stdout.print "projecthome => #{projecthome}\n"

  # TODO FIX unsafe
  # make a copy to tmp for any case untiel next reboot of host
  self.copy_file("#{userdir}/.ssh/known_hosts","/tmp/known_hosts_before_clean_up")

  File.open("#{projecthome}/\.vagrant/known_hosts_add_by_vagrant", "r") do |file_handle|
    file_handle.each_line do |server|
      # delete kez
        
        #sed -i '/pattern/d' filename

        # ruby replace in files
    end
  end
end

def set_keys_to_known_host(host)

  #$stdout.print "node => #{ node.inspect}\n"
  $stdout.print "host => #{host}\n"
  $stdout.print "host => #{host['name']}\n"
  id = %x(cat .vagrant/machines/#{host['name']}/virtualbox/id).chomp
  $stdout.print "id => #{id}\n"

# look fo forwarding of port 22 => ssh 
forwarding_port_plain = %x(/usr/bin/VBoxManage showvminfo #{id} -machinereadable| grep Forwarding |grep 22)

#TODO if port empty
forwarding_port=forwarding_port_plain.split(",")[3]
$stdout.print "forwarding_port => #{forwarding_port}\n"

# pick up keys
kez = %x(ssh-keyscan  -t ecdsa-sha2-nistp256 -H  -p #{forwarding_port} 127.0.0.1)
$stdout.print "kez => #{kez}\n"


# get user dir
userdir= Dir.home
$stdout.print "userdir => #{userdir}\n"


# add kez to cat ~/.ssh/known_hosts

open("#{userdir}/.ssh/known_hosts", 'a') do |f|
  f << "#{kez}"
end

projecthome=Dir.getwd
$stdout.print "projecthome => #{projecthome}\n"

# save added keys
# a+ - Open a file for reading and appending. The file is created if it does not exist. 
open("#{projecthome}/\.vagrant/known_hosts_add_by_vagrant", 'a+') do |f|
  f << "#{kez}"
end

end 




# }}}


# from here
# config.vm.usable_port_range = 2200..2999
# https://github.com/frapposelli/vagrant-vcloud/wiki/Increase-vagrant-default-port-range-for-larger-deploymentsy

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.ssh.insert_key = true
  config.ssh.paranoid = true
  hosts.each do |host|
    config.vm.define host['name'] do |node|
      node.vm.box = host['box'] ||= DEFAULT_BASE_BOX
      node.vm.box_url = host['box_url'] if host.key? 'box_url'

      node.vm.hostname = host['name']
      node.vm.network :private_network, network_options(host)
      
      
      # TODO old set_keys_to_known_host(node, host)
      # custom_synced_folders(node.vm, host)
      # shell_provisioners_always(node.vm, host)
      # forwarded_ports(node.vm, host)


      node.vm.provider :virtualbox do |vb|
        # WARNING: if the name of the current directory is the same as the
        # host name, this will fail.
        vb.customize ['modifyvm', :id, '--groups', PROJECT_NAME]
        #$stdout.print "vb => #{vb.inspect}\n"
        
      end
      set_keys_to_known_host(host)
      #provision_ansible(config, host)
    end
  end
end

# -*- mode: ruby -*-
# vi: ft=ruby :
