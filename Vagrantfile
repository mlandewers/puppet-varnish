Vagrant.require_version ">= 1.6.5"

# ===========================
# VARIABLES + BOX DEFINITIONS
# ===========================

SSH_BASE_PORT  = 2610
VARNISH_BASE_PORT = 6100
PUPPET_VERSION = "4.10.6"

BOXES = [
  { name: "debian7",  box: "debian/wheezy64", version: "7.11.2" },
  { name: "debian8",  box: "debian/jessie64", version: "8.9.0" },
  { name: "ubuntu14", box: "ubuntu/trusty64", version: "20170810.0.0" },
  { name: "ubuntu16", box: "ubuntu/xenial64", version: "20170811.0.0" },
  { name: "centos6",  box: "centos/6", version: "1707.01" },
  { name: "centos7",  box: "centos/7", version: "1707.01" }
]

MODULES = [
  # Module dependencies
  { name: "puppetlabs-stdlib", version: "4.17.1" },
  { name: "puppetlabs-apt", version: "2.4.0" },
  { name: "stahnma-epel", version: "1.2.2" },
  { name: "puppet-selinux", version: "1.3.0" },
  # Test dependencies
  { name: "puppetlabs-concat", version: "4.0.1" },
  { name: "puppet-nginx", git: "https://github.com/voxpupuli/puppet-nginx.git" }
]

# ==============
# VAGRANT CONFIG
# ==============

unless Vagrant.has_plugin?("vagrant-puppet-install")
  raise 'vagrant-puppet-install is not installed!'
end

Vagrant.configure("2") do |config|

  local_username ||= `whoami`.strip

  # = Actually do some work
  BOXES.each_with_index do |definition,idx|

    name = definition[:name]
    ip = 254 - idx

    config.puppet_install.puppet_version = PUPPET_VERSION
    config.vm.define name, autostart: false do |c|

      # == Basic box setup
      c.vm.box         = definition[:box]
      c.vm.box_version = definition[:version] unless definition[:version].nil?
      c.vm.hostname    = "#{local_username}-varnish-vagrant-#{name}"
      c.vm.network :private_network, ip: "10.0.254.#{ip}"

      # == Shared folder
      if Vagrant::Util::Platform.darwin?
        config.vm.synced_folder ".", "/vagrant", nfs: true
        c.nfs.map_uid = Process.uid
        c.nfs.map_gid = Process.gid
      else
        c.vm.synced_folder ".", "/vagrant", type: "nfs"
      end

      # == Disable vagrant's default SSH port, then configure our override
      new_ssh_port = SSH_BASE_PORT + idx
      c.vm.network :forwarded_port, guest: 22, host: 2222, id: "ssh", disabled: "true"
      c.ssh.port = new_ssh_port
      c.vm.network :forwarded_port, guest: 22, host: new_ssh_port

      # == Add forwarded port for Varnish (8080)
      new_varnish_port = VARNISH_BASE_PORT + idx
      c.vm.network :forwarded_port, guest: 6081, host: new_varnish_port

      # == Set resources if configured
      c.vm.provider "virtualbox" do |v|
        v.name = "puppet_varnish_#{name}"
        v.memory = definition[:memory] unless definition[:memory].nil?
        v.cpus = definition[:cpus] unless definition[:cpus].nil?
      end

      # == Install git ... with Puppet!
      c.vm.provision :shell, :inline => "/opt/puppetlabs/bin/puppet resource package git ensure=present"

      # == Install modules
      MODULES.each do |mod|
        if mod[:git].nil?
          if mod[:version].nil?
            mod_version = ''
          else
            mod_version = " --version #{mod[:version]}"
          end
          c.vm.provision :shell, :inline => "/opt/puppetlabs/bin/puppet module install #{mod[:name]}#{mod_version}"
        else
          mod_name = mod[:name].split('-').last
          c.vm.provision :shell, :inline => "if [ ! -d /etc/puppetlabs/code/environments/production/modules/#{mod_name} ]; then git clone #{mod[:git]} /etc/puppetlabs/code/environments/production/modules/#{mod_name}; fi"
        end
      end
      c.vm.provision :shell, :inline => "if [ ! -L /etc/puppetlabs/code/environments/production/modules/varnish ]; then ln -s /vagrant /etc/puppetlabs/code/environments/production/modules/varnish; fi"

      # == Finally, run Puppet!
      c.vm.provision :shell, :inline => "STDLIB_LOG_DEPRECATIONS=false /opt/puppetlabs/puppet/bin/puppet apply --verbose --show_diff /vagrant/tests/init.pp"
    end
  end
end