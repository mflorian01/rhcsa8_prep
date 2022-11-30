VAGRANTFILE_API_VERSION = "2"
VAGRANT_DISABLE_VBOXSYMLINKCREATE = "1"
file_to_disk1 = './disk-0-1.vdi'
file_to_disk2 = './disk-0-2.vdi'
$script = <<-'SCRIPT'
result=$(awk '/^PasswordAuthentication/ { print $2 }' /etc/ssh/sshd_config)
if [[ ${result} == "no" ]]; then
    sed -i '/^PasswordAuthentication/s/no/yes/g' /etc/ssh/sshd_config
    systemctl reload sshd
fi
SCRIPT
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # Use same SSH key for each machine
  config.ssh.insert_key = false
  config.vm.box_check_update = false

  # Repo Configuration
  config.vm.define "repo" do |repo|
    repo.vm.box = "mflorian/rhel8repo"
    repo.vm.hostname = "repo.example.com"
    repo.vm.provision :shell, :inline => "sudo rm -f /EMPTY;", run: "always"
    repo.vm.provision :shell, :inline => $script, run: "always"
    repo.vm.provision :shell, :inline => "yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm -y", run: "once"
    repo.vm.provision :shell, :inline => "sudo yum install -y sshpass python3-pip python3-devel httpd sshpass vsftpd createrepo", run: "once"
    repo.vm.provision :shell, :inline => " python3 -m pip install -U pip ; python3 -m pip install pexpect; python3 -m pip install ansible", run: "once"
    repo.vm.synced_folder "./vagrant", "/vagrant", type: "rsync", rsync__exclude: [".git/", ".github"]
    repo.vm.network "private_network", ip: "192.168.55.149"

    repo.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      # vb.memory = "2048"
      vb.name = "repo"
      vb.cpus = 1
    end
  end

  # Server 2 Configuration
  config.vm.define "server2" do |server2|
    server2.vm.box = "mflorian/rhel8node"
    server2.vbguest.auto_update = false
    server2.vm.provision :shell, :inline => "sudo cp /vagrant/local.repo /etc/yum.repos.d/", run: "once"
    server2.vm.provision :shell, :inline => $script, run: "always"
    server2.vm.hostname = "server2.example.com"
    server2.vm.network "private_network", ip: "192.168.55.151"
    server2.vm.network "private_network", ip: "192.168.55.175"
    server2.vm.network "private_network", ip: "192.168.55.176"
    server2.vm.synced_folder "./vagrant", "/vagrant", type: "rsync", rsync__exclude: [".git/", ".github"]
    server2.vm.provider "virtualbox" do |vb|
      
      unless File.exist?("CONTROLLER_CREATED.txt")
        vb.customize ['storagectl', :id, '--name', 'SATA Controller', '--add', 'sata', '--portcount', 2]
      end

      unless File.exist?(file_to_disk1)
        vb.customize ['createhd', '--filename', file_to_disk1, '--variant', 'Standard', '--size', 4 * 1024]
      end

      unless File.exist?(file_to_disk2)
        vb.customize ['createhd', '--filename', file_to_disk2, '--variant', 'Standard', '--size', 4 * 1024]
      end

      vb.memory = "1024"
      vb.name = "server2"
      vb.cpus = 1
      vb.customize ['storageattach', :id,  '--storagectl', 'SATA Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', file_to_disk1]
      vb.customize ['storageattach', :id,  '--storagectl', 'SATA Controller', '--port', 2, '--device', 0, '--type', 'hdd', '--medium', file_to_disk2]
    end

    server2.vm.provision :ansible_local do |ansible|
      ansible.playbook = "/vagrant/playbooks/server2.yml"
      ansible.install = false
      ansible.compatibility_mode = "2.0"
      ansible.inventory_path = "/vagrant/inventory"
      ansible.config_file = "/vagrant/ansible.cfg"
      ansible.limit = "all"
    end

    server2.trigger.after :up do |trigger|
      unless File.exist?("CONTROLLER_CREATED.txt")
        trigger.info = "Running command on host"
        # Uncomment next line if you are running vagrant from a Windows host
        trigger.run = {inline: "echo $null > CONTROLLER_CREATED.txt"}
        # Uncomment next line if you are running vagrant from Unix-like host
        # trigger.run = {inline: "echo > CONTROLLER_CREATED.txt"}
      end
    end
    server2.trigger.after :destroy do |trigger|
      trigger.info = "Running command on host"
      # Uncomment next line if you are running vagrant from a Windows host
      trigger.run = {inline: "del CONTROLLER_CREATED.txt"}
      # Uncomment next line if you are running vagrant from Unix-like host
      # trigger.run = {inline: "rm -f CONTROLLER_CREATED.txt"}
    end
    # server2.vm.provision :shell, :inline => "reboot", run: "always"
  end

  # Server 1 Configuration
  config.vm.define "server1" do |server1|
    server1.vm.box = "mflorian/rhel8node"
    server1.vbguest.auto_update = false
    server1.vm.provision :shell, :inline => "sudo cp /vagrant/local.repo /etc/yum.repos.d/", run: "once"
    server1.vm.provision :shell, :inline => $script, run: "always"
    server1.vm.synced_folder "./vagrant", "/vagrant", type: "rsync", rsync__exclude: [".git/", ".github"]
    server1.vm.hostname = "server1.example.com"
    server1.vm.network "private_network", ip: "192.168.55.150"
    server1.vm.provider :virtualbox do |vb|
      vb.customize ['modifyvm', :id,'--memory', '1024']
      vb.name = "server1"
      vb.cpus = 1
    end

    server1.vm.provision :ansible_local do |ansible|
      ansible.playbook = "/vagrant/playbooks/server1.yml"
      ansible.install = false
      ansible.compatibility_mode = "2.0"
      ansible.inventory_path = "/vagrant/inventory"
      ansible.config_file = "/vagrant/ansible.cfg"
      ansible.limit = "all"
    end
    # server1.vm.provision :shell, :inline => "reboot", run: "always"
  end

  # LDAP / NFS / Samba server
  config.vm.define "ldap" do |ldap|
    ldap.vm.box = "mflorian/centos7"
    # Working version, uncomment this if current version is not working
    # ldap.vm.box_version = "4.2.2"
    ldap.vbguest.auto_update = false
    ldap.vm.provision :shell, :inline => $script, run: "always"
    ldap.vm.provision :shell, :inline => "yum -y install ansible libsemanage-python policycoreutils-python", run: "once"
    ldap.vm.synced_folder "./vagrant", "/vagrant", type: "rsync", rsync__exclude: [".git/", ".github"]
    ldap.vm.hostname = "ldap.example.com"
    ldap.vm.network "private_network", ip: "192.168.55.152"
    ldap.vm.provider :virtualbox do |vb|
      vb.customize ['modifyvm', :id,'--memory', '1024']
      vb.name = "ldap"
      vb.cpus = 1
    end


    ldap.vm.provision :ansible_local do |ansible|
      ansible.playbook = "/vagrant/playbooks/ldap.yml"
      ansible.install = false
      ansible.compatibility_mode = "2.0"
      ansible.inventory_path = "/vagrant/inventory"
      ansible.config_file = "/vagrant/ansible.cfg"
      ansible.limit = "all"
    end
    # ldap.vm.provision :shell, :inline => "reboot", run: "always"
  end

  # LDAP client Centos8
  config.vm.define "ldapclient8" do |ldapclient8|
    ldapclient8.vbguest.auto_update = false
    ldapclient8.vm.box = "mflorian/rhel8node"
    ldapclient8.vm.provision "file", source: "./vagrant/local.repo", destination: "/home/vagrant/local.repo", run: "always"
    ldapclient8.vm.provision :shell, :inline => "sudo cp /home/vagrant/local.repo /etc/yum.repos.d/", run: "always"
    ldapclient8.vm.provision :shell, :inline => "sudo rm -f /EMPTY;", run: "always"
    ldapclient8.vm.provision :shell, :inline => "yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm -y; sudo yum install -y sshpass python3-pip python3-devel httpd sshpass vsftpd createrepo", run: "always"
    ldapclient8.vm.provision :shell, :inline => " python3 -m pip install -U pip ; python3 -m pip install pexpect; python3 -m pip install ansible", run: "always"
    ldapclient8.vm.provision :shell, :inline => $script, run: "always"
    # ldapclient8.vm.provision :shell, :inline => "yum -y install ansible", run: "always"
    ldapclient8.vm.synced_folder "./vagrant", "/vagrant", type: "rsync", rsync__exclude: [".git/", ".github"]
    ldapclient8.vm.hostname = "ldapclient8.example.com"
    ldapclient8.vm.network "private_network", ip: "192.168.55.171"
    ldapclient8.vm.provider :virtualbox do |vb|
      vb.customize ['modifyvm', :id,'--memory', '1024']
      vb.name = "ldapclient8"
      vb.cpus = 1
    end

    ldapclient8.vm.provision :ansible_local do |ansible|
      ansible.playbook = "/vagrant/playbooks/ldapclient8.yml"
      ansible.install = false
      ansible.compatibility_mode = "2.0"
      ansible.inventory_path = "/vagrant/inventory"
      ansible.config_file = "/vagrant/ansible.cfg"
      ansible.limit = "all"
    end
    # ldapclient8.vm.provision :shell, :inline => "reboot", run: "always"
  end
end  
