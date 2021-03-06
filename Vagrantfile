# -*- mode: ruby -*-
# # vi: set ft=ruby :

require 'fileutils'

Vagrant.require_version ">= 1.9.0"

SYSCTL_CONF_PATH = File.join(File.dirname(__FILE__), "src/vagrant/sysctl.conf")
ORACLE_JDK8_PATH = File.join(File.dirname(__FILE__), "src/vagrant/jdk-8u121-linux-x64.tar.gz")
JCE_POLICY8_PATH = File.join(File.dirname(__FILE__), "src/vagrant/jce_policy-8.zip")
APP_PACKAGE_PATH = File.join(File.dirname(__FILE__), "target/demo-quasar-0.0.1.KUANGHENG-SNAPSHOT-distribution.tar.gz")
INIT_SCRIPT_PATH = File.join(File.dirname(__FILE__), "src/vagrant/init-script.sh")
CONFIG = File.join(File.dirname(__FILE__), "config.rb")

# Defaults for config options defined in CONFIG
$enable_serial_logging = false
$forwarded_ports = {}
$shared_folders = {}
$vb_cpuexecutioncap = 100
$vm_cpus = 4
$vm_gui = false
$vm_memory = 4096

if File.exist?(CONFIG)
  require CONFIG
end

# Use old vb_xxx config variables when set
def vm_cpus
  $vb_cpus.nil? ? $vm_cpus : $vb_cpus
end

def vm_gui
  $vb_gui.nil? ? $vm_gui : $vb_gui
end

def vm_memory
  $vb_memory.nil? ? $vm_memory : $vb_memory
end

# see: https://superuser.com/questions/701735/run-script-on-host-machine-during-vagrant-up
system("
    if [ #{ARGV[0]} = 'up' ]; then
        echo 'vagrant up and execute script'
        ./src/vagrant/download_oracle_jdk.sh

        (cd .. && mvn -Dmaven.test.skip=true clean install)
        mvn -Dmaven.test.skip=true clean package assembly:single
    fi
")

Vagrant.configure("2") do |config|
  #config.vm.box = "ubuntu/trusty64"
  # We don't use ubuntu/xenial64 because of https://bugs.launchpad.net/cloud-images/+bug/1569237
  #config.vm.box = "ubuntu/xenial64"
  config.vm.box = "centos/7"

  # always use Vagrants insecure key
  config.ssh.insert_key = false
  # forward ssh agent to easily ssh into the different machines
  config.ssh.forward_agent = true

  config.vm.provider :virtualbox do |v|
    # On VirtualBox, we don't have guest additions or a functional vboxsf
    # in CoreOS, so tell Vagrant that so it can be smarter.
    v.check_guest_additions = false
    v.functional_vboxsf     = false
  end

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  config.vm.define vm_name = "demo-quasar" do |config|
    config.vm.hostname = vm_name

    if $enable_serial_logging
      logdir = File.join(File.dirname(__FILE__), "log")
      FileUtils.mkdir_p(logdir)

      serialFile = File.join(logdir, "%s-serial.txt" % vm_name)
      FileUtils.touch(serialFile)

      ["vmware_fusion", "vmware_workstation"].each do |vmware|
        config.vm.provider vmware do |v, override|
          v.vmx["serial0.present"] = "TRUE"
          v.vmx["serial0.fileType"] = "file"
          v.vmx["serial0.fileName"] = serialFile
          v.vmx["serial0.tryNoRxLoss"] = "FALSE"
        end
      end

      config.vm.provider :virtualbox do |vb, override|
        vb.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
        vb.customize ["modifyvm", :id, "--uartmode1", serialFile]
      end
    end

    $forwarded_ports.each do |guest, host|
      config.vm.network "forwarded_port", guest: guest, host: host, auto_correct: true
    end

    ["vmware_fusion", "vmware_workstation"].each do |vmware|
      config.vm.provider vmware do |v|
        v.gui = vm_gui
        v.vmx['memsize'] = vm_memory
        v.vmx['numvcpus'] = vm_cpus
      end
    end
    
    config.vm.provider :virtualbox do |vb|
      vb.gui = vm_gui
      vb.memory = vm_memory
      vb.cpus = vm_cpus
      vb.customize ["modifyvm", :id, "--cpuexecutioncap", "#{$vb_cpuexecutioncap}"]
    end

    ip = "192.168.50.50"
    config.vm.network :private_network, ip: ip
    #config.vm.network "private_network",
    #  ip: "fde4:8dba:82e1::c4",
    #  netmask: "96"

    if File.exist?(SYSCTL_CONF_PATH)
      config.vm.provision :file, :source => "#{SYSCTL_CONF_PATH}", :destination => "/tmp/sysctl.conf"
      config.vm.provision :shell, :inline => "if [[ -f /tmp/sysctl.conf ]]; then mv /tmp/sysctl.conf /etc/sysctl.conf; sysctl -p; fi", :privileged => true
    end
    if File.exist?(ORACLE_JDK8_PATH)
      config.vm.provision :file, :source => "#{ORACLE_JDK8_PATH}", :destination => "/home/vagrant/jdk8.tar.gz"
    end
    if File.exist?(JCE_POLICY8_PATH)
      config.vm.provision :file, :source => "#{JCE_POLICY8_PATH}", :destination => "/home/vagrant/jce_policy-8.zip"
    end
    if File.exist?(APP_PACKAGE_PATH)
      config.vm.provision :file, :source => "#{APP_PACKAGE_PATH}", :destination => "/home/vagrant/application.tar.gz"
    end
    if File.exist?(INIT_SCRIPT_PATH)
      config.vm.provision :file, :source => "#{INIT_SCRIPT_PATH}", :destination => "/tmp/vagrantfile-init-script.sh"
      config.vm.provision :shell, :inline => "/tmp/vagrantfile-init-script.sh", :privileged => true
    end

  end
end
