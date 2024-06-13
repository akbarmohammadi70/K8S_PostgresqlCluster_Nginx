
NUM_NODE = 6
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-22.04-arm64"
  config.vm.box_check_update = false
  (1..NUM_NODE).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.provider "vmware_desktop" do |vmware|
        vmware.cpus = 2
        vmware.memory = 2048
        vmware.gui = true
        vmware.vmx["ethernet0.virtualdev"] = "vmxnet3"
      end
      node.vm.hostname = "node#{i}"
      node.vm.network "forwarded_port", guest: 22, host: "#{2720 + i}"
      node.vm.network "private_network", ip: "192.168.61.#{i + 9}"
      node.vm.provision "shell", inline: <<-SHELL
        mkdir -p /home/vagrant/.ssh
        echo "$(cat /Users/am/.ssh/id_rsa.pub)" >> /home/vagrant/.ssh/authorized_keys
        chown -R vagrant:vagrant /home/vagrant/.ssh
        chmod 700 /home/vagrant/.ssh
        chmod 600 /home/vagrant/.ssh/authorized_keys
      SHELL
      node.vm.provision "setup-hosts", :type => "shell", :path => "scripts/setup-hosts.sh" do |s|
        s.args = ["eth0"]
      end
    end
  end
end
