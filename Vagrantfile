# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Ubuntu 20.04
  config.vm.box = "ubuntu/focal64"

  # You can change these names & IPs to whatever you like
  vm_definitions = [
    { name: "ubuntu-vm1", ip: "192.168.1.111" }
    # { name: "ubuntu-vm2", ip: "192.168.1.112" },
    # { name: "ubuntu-vm3", ip: "192.168.1.113" }
  ]

  vm_definitions.each do |vm|
    config.vm.define vm[:name] do |node|
      node.vm.hostname = vm[:name]

      # Bridged network on the same subnet as your host
      node.vm.network "public_network",
        ip:     vm[:ip],
        bridge: "Intel(R) Dual Band Wireless-AC 8265"

      # Provider-specific VM settings
      node.vm.provider "virtualbox" do |vb|
        vb.name   = vm[:name]
        vb.memory = 3072
        vb.cpus   = 2
      end

      # As root
      node.vm.provision "shell", privileged: true, inline: <<-SHELL
        # Update & upgrade
        apt-get update -y && apt-get upgrade -y  
        # Install Apache & SSH
        apt-get install -y apache2 openssh-client openssh-server
        # Enable SSH password auth & restart
        sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
        sed -i 's/^#*ChallengeResponseAuthentication.*/ChallengeResponseAuthentication yes/' /etc/ssh/sshd_config
        systemctl restart ssh
        # Set vagrant password
        echo 'vagrant:vagrant' | chpasswd
        # Allow SSH through firewall
        ufw allow ssh
        ufw --force enable
      SHELL
      # As vagrant
      node.vm.provision "shell", privileged: false, inline: <<-SHELL
        # Update & upgrade
        sudo apt-get update -y && sudo apt-get upgrade -y
        pwd
        # Turn off swap now and comment it out in fstab
        sudo swapoff -a
        sudo sed -i '/swap/s/^/#/' /etc/fstab

        # === HERE: Containerd kernel modules ===
        sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
        # Load them right now (so you don’t have to reboot)
        sudo modprobe overlay
        sudo modprobe br_netfilter
        # Network config
        echo "net.bridge.bridge-nf-call-ip6tables = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
        echo "net.bridge.bridge-nf-call-iptables = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
        echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
        sudo sysctl --system
        # Add package to docker
        sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
        sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
        sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
        # Install containerd
        sudo apt update -y
        sudo apt install -y containerd.io
        containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
        sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
        sudo systemctl restart containerd
        sudo systemctl enable containerd
        # Add k8s repository
        sudo mkdir -p /etc/apt/keyrings
        echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
        curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
        # Install k8s packages
        sudo apt update -y
        sudo apt install -y kubelet kubeadm kubectl
        sudo apt-mark hold kubelet kubeadm kubectl
        # Reset cluster if it is exits
        sudo kubeadm reset -f
        sudo rm -rf /var/lib/etcd
        sudo rm -rf /etc/kubernetes/manifests/*
      SHELL
    end
  end
end
