# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-22.04"
  config.vm.box_version = "202401.31.0"

  config.vm.synced_folder ".", "/vagrant"

  config.vm.network "private_network", type: "dhcp"
  # config.vm.network "private_network", ip: "192.168.33.10"
  # config.vm.network "public_network", type: "dhcp"

  config.vm.provider "virtualbox" do |vb|
    vb.cpus = "2"
    # vb.cpus = "4"
    vb.memory = "2048"
    # vb.memory = "4096"
  end

  config.vm.provision "shell", inline: <<-SHELL
    # タイムゾーンの設定
    apt-get install -y tzdata
    ln -fs /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
    dpkg-reconfigure -f noninteractive tzdata

    # IP アドレス確認
    ip addr show eth1 | grep 'inet ' | awk '{print $2}' | cut -d/ -f1
  SHELL

end
