# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = '2'

NMONS   = 1
NOSDS   = 1
SUBNET  = '192.168.101'

ansible_provision = proc do |ansible|
  ansible.playbook = 'site.yml'
  # Note: Can't do ranges like mon[0-2] in groups because
  # these aren't supported by Vagrant, see
  # https://github.com/mitchellh/vagrant/issues/3539
  ansible.groups = {
    'mons' => (0..NMONS - 1).map { |j| "mon#{j}" },
    'osds' => (0..NOSDS - 1).map { |j| "osd#{j}" },
  }

  # In a production deployment, these should be secret
  ansible.extra_vars = {
    fsid: '10c95f01-2dd2-4863-affa-60c4eafcd8d2',
    monitor_secret: 'AQBNTxZRWId7JxAA/Ac4ToR7ZfNdOGDSToGHpA=='
  }
  ansible.limit = 'all'
end

def create_vmdk(name, size)
  dir = Pathname.new(__FILE__).expand_path.dirname
  path = File.join(dir, '.vagrant', name + '.vmdk')
  `vmware-vdiskmanager -c -s #{size} -t 0 -a scsi #{path} \
   2>&1 > /dev/null` unless File.exist?(path)
end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
config.vm.box = 'TryCeph-ksingh'
config.vm.box_url = 'https://www.dropbox.com/s/man87m5ywc5je2e/package.box?dl=1'
#config.vm.box_url = 'file://package.box'
config.ssh.pty = true
  config.ssh.insert_key = false # workaround for https://github.com/mitchellh/vagrant/issues/5048


  (0..NMONS - 1).each do |i|
    config.vm.define "mon#{i}" do |mon|
      mon.vm.hostname = "mon#{i}"
      mon.vm.network :private_network, ip: "#{SUBNET}.1#{i}"
      mon.vm.network "forwarded_port", guest: 5000, host: 8080
      mon.vm.provider :virtualbox do |vb|
        vb.customize ['modifyvm', :id, '--memory', '192']
      end
      mon.vm.provider :vmware_fusion do |v|
        v.vmx['memsize'] = '192'
      end
    end
  end

  (0..NOSDS - 1).each do |i|
    config.vm.define "osd#{i}" do |osd|
      osd.vm.hostname = "osd#{i}"
      osd.vm.network :private_network, ip: "#{SUBNET}.10#{i}"
      osd.vm.network :private_network, ip: "#{SUBNET}.20#{i}"
      osd.vm.provider :virtualbox do |vb|
      (0..1).each do |d|
          vb.customize ['createhd',
                        '--filename', "disk-#{i}-#{d}",
                        '--size', '11000']
          vb.customize ['storageattach', :id,
                        '--storagectl', 'SATA',
                        '--port', 3 + d,
                        '--device', 0,
                        '--type', 'hdd',
                        '--medium', "disk-#{i}-#{d}.vdi"]
        end
	vb.customize ['modifyvm', :id, '--memory', '512']

      end
      osd.vm.provider :vmware_fusion do |v|
        (0..1).each do |d|
          v.vmx["scsi0:#{d + 1}.present"] = 'TRUE'
          v.vmx["scsi0:#{d + 1}.fileName"] =
            create_vmdk("disk-#{i}-#{d}", '11000MB')
        end
        v.vmx['memsize'] = '192'
      end

      # Run the provisioner after the last machine comes up
      osd.vm.provision 'ansible', &ansible_provision if i == (NOSDS - 1)
    end
  end
end
