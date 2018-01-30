# One Vagrantfile to rule them all!
#
# This is a generic Vagrantfile that can be used without modification in
# a variety of situations. Hosts and their properties are specified in
# `vagrant-hosts.yml`.
#
# See https://github.com/bertvv/ansible-skeleton/ for details

require 'yaml'
require 'open3'
require 'log4r'

# define logger
logger = Log4r::Logger.new('vagrant::vagrantfile')
logger.debug 'enable logger debug'
logger.info 'enable logger info'

# set default LC_ALL for all BOXES
ENV['LC_ALL'] = 'en_US.UTF-8'

# Set your default base box here
DEFAULT_BASE_BOX = 'none/none_set_default_box_name'.freeze

VAGRANTFILE_API_VERSION = '2'.freeze

VAGRANT_VERSION = '2.0.1'.freeze

PROJECT_NAME = '/' + File.basename(Dir.getwd)

PROJECT_HOME = Dir.getwd
logger.debug "PROJECT_HOME => #{PROJECT_HOME}\n"

# set user dir
USERDIR = Dir.home
logger.debug "USERDIR => #{USERDIR}\n"

# set path to known_hosts
known_hosts = "#{USERDIR}/\.ssh/known_hosts"
logger.debug "known_hosts => #{known_hosts}\n"

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
          'ansible/site.yml'
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
#
# example:
# shell_always:
# - cmd: php /srv/google-dev/bin/console server:start 192.168.52.25:8080 --force
def shell_provisioners_always(vm, host)
  if host.key?('shell_always')
    scripts = host['shell_always']

    scripts.each do |script|
      vm.provision 'shell', inline: script['cmd'], run: 'always'
    end
  end
end

# TODO: old delete export VAGRANTS_HOST='custom-vagrant-hosts.yml'

# }}}

# Adds forwarded ports to your vagrant machine
# so they are available from your phone
#
# example:
#  forwarded_ports:
#    - guest: 88
#      host: 8080
def forwarded_ports(vm, host)
  if host.key?('forwarded_ports')
    ports = host['forwarded_ports']

    ports.each do |port|
      vm.network 'forwarded_port', guest: port['guest'], host: port['host']
    end
  end
end

# copy file
def self.copy_file(src, dest)
  # w - Create an empty file for writing.
  File.open(dest, 'w') { |f| f.write(File.read(src)) }
end

def remove_kez_added_by_vagrant(host, known_hosts, logger)
  info "Remove key from VM #{host['name']}"

  known_hosts_add_by_vagrant = "#{PROJECT_HOME}/\.vagrant/machines/#{host['name']}/known_hosts_add_by_vagrant"

  if File.exist?(known_hosts_add_by_vagrant)

    # TODO: make path/file global
    File.open(known_hosts_add_by_vagrant, 'r') do |file_handle|
      file_handle.each_line do |server_kez|
        if server_kez.empty?
          logger.warn "No Key found for VM #{host['name']}"
          logger.warn 'Please remove by hand'
        else
          # escaped string
          server_kez = server_kez.gsub!('|', '\\|')
          server_kez = server_kez.gsub!('/', '\\/')
          # delete kez
          info "Delete Key #{server_kez} from file ~/.ssh/known_hosts "
          command = "/bin/sed -i '/#{server_kez.chomp}/d' #{known_hosts}"
          logger.debug "command =>#{command}"
          # from here
          # https://stackoverflow.com/questions/6338908/ruby-difference-between-exec-system-and-x-or-backticks
          Open3.popen3(command) do |_stdin, stdout, stderr, _thread|
            # TODO: unused pid = thread.pid
            # TODO debug remove
            info stdout.read.chomp
            info stderr.read.chomp
          end
        end
      end
    end
    # remove kez store
    FileUtils.remove_file(known_hosts_add_by_vagrant, force = false)
    logger.info "File #{known_hosts_add_by_vagrant} was deleted"
  else
    logger.info "File #{known_hosts_add_by_vagrant} already deleted"
  end
end

def set_keys_to_known_host(host, known_hosts, logger)
  logger.debug "host => #{host['name']}\n"

  id = `cat .vagrant/machines/#{host['name']}/virtualbox/id`.chomp
  logger.debug "id => #{id}\n"

  # look fo forwarding of port 22 => ssh
  forwarding_port_plain = `/usr/bin/VBoxManage showvminfo #{id} -machinereadable|grep Forwarding |grep 22`

  # TODO: if port empty
  forwarding_port = forwarding_port_plain.split(',')[3]
  logger.debug "forwarding_port => #{forwarding_port}\n"

  # pick up keys
  kez = `ssh-keyscan  -t ecdsa-sha2-nistp256 -H -p #{forwarding_port} 127.0.0.1`
  logger.debug "kez => #{kez}\n"

  # add kez to cat ~/.ssh/known_hosts
  open(known_hosts.to_s, 'a+') do |file_handler|
    file_handler << kez.to_s
  end

  # save added keys
  # a+ - Open a file for reading and appending. The file is created if it does not exist.
  open("#{PROJECT_HOME}/\.vagrant/machines/#{host['name']}/known_hosts_add_by_vagrant", 'a+') do |file_handler|
    file_handler << kez.to_s
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
    # register triggers
    [:up].each do |cmd|
      config.trigger.after cmd, stdout: true, force: true, vm: host['name'] do
        info " register triggers after  #{cmd} for  #{host['name']}"
        logger.debug " register triggers host['name'] #{host['name']}\n"
        logger.debug "known_host => #{known_hosts}\n"
        set_keys_to_known_host(host, known_hosts, logger)
      end
    end

    # TODO: what is about  halt
    %i[destroy].each do |cmd|
      config.trigger.after cmd, stdout: true, force: true, vm: host['name'] do
        info " register triggers after #{cmd} for VM #{host['name']}"
        logger.debug " register triggers host['name'] #{host['name']}\n"
        logger.debug "known_host => #{known_hosts}\n"
        remove_kez_added_by_vagrant(host, known_hosts, logger)
      end
    end

    config.vm.define host['name'] do |node|
      node.vm.box = host['box'] ||= DEFAULT_BASE_BOX
      node.vm.box_url = host['box_url'] if host.key? 'box_url'
      node.vm.hostname = host['name']
      node.vm.network :private_network, network_options(host)

      # disable install rsync
      config.vm.synced_folder '.', '/vagrant', disabled: true

      # set_keys_to_known_host(host)
      # custom_synced_folders(node.vm, host)
      # shell_provisioners_always(node.vm, host)
      # forwarded_ports(node.vm, host)

      node.vm.provider :virtualbox do |vb|
        # WARNING: if the name of the current directory is the same as the
        # host name, this will fail.
        vb.customize ['modifyvm', :id, '--groups', PROJECT_NAME]
        # logger.debug "vb => #{vb.inspect}\n"
      end

      # set_keys_to_known_host(host)
      # provision_ansible(config, host)
    end
  end
end

# -*- mode: ruby -*-
# vi: ft=ruby :
