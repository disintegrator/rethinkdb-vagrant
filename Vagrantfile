# -*- mode: ruby -*-
# # vi: set ft=ruby :

require 'fileutils'

Vagrant.require_version ">= 1.6.0"

# Defaults for config options defined in CONFIG
$num_instances = 3
$instance_name_prefix = "rethink"
$vm_gui = false
$vm_memory = 1024
$vm_cpus = 2
$forwarded_ports = {
  29015 => 29015,
  28015 => 28015,
  8080 => 8080
}

# Use old vb_xxx config variables when set
def vm_gui
  $vb_gui.nil? ? $vm_gui : $vb_gui
end

def vm_memory
  $vb_memory.nil? ? $vm_memory : $vb_memory
end

def vm_cpus
  $vb_cpus.nil? ? $vm_cpus : $vb_cpus
end

Vagrant.configure("2") do |config|
  # always use Vagrants insecure key
  config.ssh.insert_key = false

  config.vm.box = "ubuntu/trusty64"
  config.vm.box_version = "~> 14.04"

  (0...$num_instances).each do |i|
    config.vm.define vm_name = "%s-%02d" % [$instance_name_prefix, i], primary: i == 0, autostart: i == 0 do |cfg|
      cfg.vm.hostname = vm_name

      cfg.vm.provider :virtualbox do |vb, override|
        vb.customize ['modifyvm', :id, '--memory', '512', '--ioapic', 'on']
      end

      $forwarded_ports.each do |guest, host|
        cfg.vm.network "forwarded_port", guest: guest, host: host, auto_correct: i != 0
      end

      ["vmware_fusion", "vmware_workstation"].each do |vmware|
        cfg.vm.provider vmware do |v|
          v.gui = vm_gui
          v.vmx['memsize'] = vm_memory
          v.vmx['numvcpus'] = vm_cpus
        end
      end

      cfg.vm.provider :virtualbox do |vb|
        vb.gui = vm_gui
        vb.memory = vm_memory
        vb.cpus = vm_cpus
      end

      ip = "10.10.10.#{10 + i}"
      cfg.vm.network "private_network", ip: ip, virtualbox__intnet: true

      joincmd = "echo Bootstrap node"
      if i > 0
        joincmd = "echo join=10.10.10.10:29015 >> /etc/rethinkdb/instances.d/default.conf"
      end

      # Provisioning
      config.vm.provision :shell do |sh|
        sh.inline = <<-EOF
          export DEBIAN_FRONTEND=noninteractive;
          # Add RethinkDB Source
          apt-key adv --fetch-keys http://download.rethinkdb.com/apt/pubkey.gpg 2>&1;
          echo "deb http://download.rethinkdb.com/apt $(lsb_release -sc) main" > /etc/apt/sources.list.d/rethinkdb.list;
          apt-get update --assume-yes;
          # RethinkDB Install & Setup
          apt-get install --assume-yes rethinkdb;
          sed -e 's/# bind=127.0.0.1/bind=all/g' /etc/rethinkdb/default.conf.sample > /etc/rethinkdb/instances.d/default.conf;
          #{joincmd}
          rethinkdb create -d /var/lib/rethinkdb/instances.d/default 2>&1;
          service rethinkdb restart;
        EOF
      end
    end
  end
end
