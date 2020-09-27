# -*- mode: ruby -*-
# # vi: set ft=ruby :

require 'fileutils'

Vagrant.require_version ">= 2.0.0"

CONFIG = File.join(File.dirname(__FILE__), ENV['KUBESPRAY_VAGRANT_CONFIG'] || 'vagrant/config.rb')

FLATCAR_URL_TEMPLATE = "https://%s.release.flatcar-linux.net/amd64-usr/current/flatcar_production_vagrant.json"

# Uniq disk UUID for libvirt
DISK_UUID = Time.now.utc.to_i

SUPPORTED_OS = {
  "ubuntu1604"          => {box: "generic/ubuntu1604",         user: "vagrant"},
  "ubuntu1804"          => {box: "generic/ubuntu1804",         user: "vagrant"},
  "ubuntu2004"          => {box: "generic/ubuntu2004",         user: "vagrant"},
  "centos"              => {box: "centos/7",                   user: "vagrant"},
  "centos-bento"        => {box: "bento/centos-7.6",           user: "vagrant"},
  "centos8"             => {box: "centos/8",                   user: "vagrant"},
  "centos8-bento"       => {box: "bento/centos-8",             user: "vagrant"},
  "fedora31"            => {box: "fedora/31-cloud-base",       user: "vagrant"},
  "fedora32"            => {box: "fedora/32-cloud-base",       user: "vagrant"},
  "opensuse"            => {box: "bento/opensuse-leap-15.1",   user: "vagrant"},
  "opensuse-tumbleweed" => {box: "opensuse/Tumbleweed.x86_64", user: "vagrant"},
  "oraclelinux"         => {box: "generic/oracle7",            user: "vagrant"},
  "oraclelinux8"        => {box: "generic/oracle8",            user: "vagrant"},
}

if File.exist?(CONFIG)
  require CONFIG
end

$script = <<SCRIPT
    if [ -f /vagrant/.ssh/id_rsa ]; then
        /bin/cp -Rf /vagrant/.ssh/id_rsa* /home/vagrant/.ssh/
    fi
    if [ -f /vagrant/id_rsa ]; then
        /bin/cp -Rf /vagrant/id_rsa* /home/vagrant/.ssh/
    fi

    find /home/vagrant/.ssh -type f | xargs chmod -R 600
    find /home/vagrant/.ssh -type d | xargs chmod -R 700
    find /home/vagrant/.ssh -type f | xargs chown -R vagrant:vagrant
    find /home/vagrant/.ssh -type d | xargs chown -R vagrant:vagrant

    # If .gitconfig not exist in current dir, copy example from Vagrant
    if [ ! -f /vagrant/.gitconfig ]; then
        if [ -f /home/vagrant/.gitconfig ]; then
          cp /home/vagrant/.gitconfig /vagrant/.gitconfig
        else
          touch /home/vagrant/.gitconfig
        fi
    fi
    echo "cd /vagrant" >> /home/vagrant/.bashrc
    echo "export ANSIBLE_FORCE_COLOR=true" >> /home/vagrant/.bashrc
SCRIPT

$num_instances ||= 1
$instance_name_prefix ||= "vagrantbox"
$vm_gui ||= false
$vm_memory ||= 2048
$vm_cpus ||= 2
$shared_folders ||= {}
$forwarded_ports ||= {}
$subnet ||= "172.18.8"
$os ||= "ubuntu1804"
$disk_size ||= "20GB"
$playbook ||= "akeneo-box.yml"
host_vars = {}
$box = SUPPORTED_OS[$os][:box]
$inventory = "inventory/sample" if ! $inventory
$inventory = File.absolute_path($inventory, File.dirname(__FILE__))

Vagrant.configure("2") do |config|
  config.vm.box = $box
  if SUPPORTED_OS[$os].has_key? :box_url
    config.vm.box_url = SUPPORTED_OS[$os][:box_url]
  end
  config.ssh.username = SUPPORTED_OS[$os][:user]

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  # always use Vagrants insecure key
  config.ssh.insert_key = false

  if ($override_disk_size)
    unless Vagrant.has_plugin?("vagrant-disksize")
      system "vagrant plugin install vagrant-disksize"
    end
    config.disksize.size = $disk_size
  end

  (1..$num_instances).each do |i|
    config.vm.define vm_name = "%s-%01d" % [$instance_name_prefix, i] do |node|

      node.vm.hostname = vm_name
      node.vm.provider :virtualbox do |vb|
        vb.memory = $vm_memory
        vb.cpus = $vm_cpus
        vb.gui = $vm_gui
        vb.linked_clone = true
        vb.customize ["modifyvm", :id, "--vram", "8"] # ubuntu defaults to 256 MB which is a waste of precious RAM
      end

      # node.vm.synced_folder ".", "/vagrant", disabled: false, type: "rsync", rsync__args: ['--verbose', '--archive', '--delete', '-z'] , rsync__exclude: ['.git','venv']
      node.vm.synced_folder ".", "/vagrant", disabled: false, create: true, owner: "vagrant", group: "vagrant"
      $shared_folders.each do |src, dst|
        node.vm.synced_folder src, dst, type: "rsync", rsync__args: ['--verbose', '--archive', '--delete', '-z']
      end

      ip = "#{$subnet}.#{i+100}"
      node.vm.network :private_network, ip: ip

      # Disable swap for each vm
      node.vm.provision "shell", inline: "swapoff -a"

      # Disable firewalld on oraclelinux vms
      if ["oraclelinux","oraclelinux8"].include? $os
        node.vm.provision "shell", inline: "systemctl stop firewalld; systemctl disable firewalld"
      end

      host_vars[vm_name] = {
        "ip": ip,
        "local_path_provisioner_enabled": "#{$local_path_provisioner_enabled}",
        "local_path_provisioner_claim_root": "#{$local_path_provisioner_claim_root}",
        "ansible_ssh_user": SUPPORTED_OS[$os][:user]
      }

      # Only execute the Ansible provisioner once, when all the machines are up and ready.
      if i == $num_instances
        node.vm.provision "ansible" do |ansible|
          ansible.playbook = $playbook
          $ansible_inventory_path = File.join( $inventory, "hosts.ini")
          if File.exist?($ansible_inventory_path)
            ansible.inventory_path = $ansible_inventory_path
          end
          ansible.become = true
          ansible.limit = "all,localhost"
          ansible.host_key_checking = false
          ansible.raw_arguments = ["--forks=#{$num_instances}", "--flush-cache", "-e ansible_become_pass=vagrant"]
          ansible.host_vars = host_vars
        end
      end
    end
  end
  config.vm.provision "shell", inline: $script
end
