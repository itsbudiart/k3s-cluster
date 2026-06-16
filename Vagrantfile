# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-24.04"

  nodes = [
    { name: "k3s-master",   ip: "192.168.56.33", role: "server", cpus: 1, memory: 2048 },
    { name: "k3s-worker-1", ip: "192.168.56.34", role: "agent",  cpus: 1, memory: 2048 },
    { name: "k3s-worker-2", ip: "192.168.56.35", role: "agent",  cpus: 1, memory: 2048 }
  ]

  nodes.each do |node|
    config.vm.define node[:name] do |machine|
      machine.vm.hostname = node[:name]

      machine.vm.network "private_network",
        ip: node[:ip]

      if Vagrant.has_plugin?("vagrant-disksize")
        machine.disksize.size = "20GB"
      end

      machine.vm.provider "vmware_desktop" do |v|
        v.gui = false

        # VMware VM name
        v.vmx["displayName"] = "k3s-#{node[:name]}"

        # CPU and memory
        v.vmx["numvcpus"] = node[:cpus].to_s
        v.vmx["memsize"]  = node[:memory].to_s
      end

      # =========================
      # PROVISIONING
      # =========================
      machine.vm.provision "shell", inline: <<-SHELL
        echo "===> Updating Ubuntu packages..."

        sudo apt-get update -y
        sudo DEBIAN_FRONTEND=noninteractive apt-get upgrade -y

        echo "===> Installing basic packages..."

        sudo apt-get install -y \
          curl \
          wget \
          vim \
          git \
          net-tools \
          ca-certificates \
          gnupg \
          lsb-release \
          apt-transport-https

        echo "===> Done provisioning #{node[:name]}"
      SHELL
    end
  end
end