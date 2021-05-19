Vagrant.configure("2") do |config|
    config.vm.provider :virtualbox do |v|
      v.memory = 4096
      v.cpus = 2
    end
  
    config.vm.provision :shell, privileged: true, inline: $install_common_tools
  
    config.vm.define :master do |master|
      master.vm.box = "ubuntu/xenial64"
      master.vm.hostname = "master"
      master.vm.network :private_network, ip: "dhcp"
      master.vm.provision :shell, privileged: false, inline: $provision_master_node
    end
  
    %w{worker1 worker2}.each_with_index do |name, i|
      config.vm.define name do |worker|
        worker.vm.box = "ubuntu/xenial64"
        worker.vm.hostname = name
        worker.vm.network :private_network, ip: "dhcp}"
        worker.vm.provision :shell, privileged: false, inline: <<-SHELL
  sudo /vagrant/join.sh
  echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=10.0.0.#{i + 11}"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
  sudo systemctl daemon-reload
  sudo systemctl restart kubelet
  SHELL
      end
    end
  
    config.vm.provision "shell", inline: $install_multicast
  end
  
  
  Vagrant.configure("2") do |config|
    config.vm.provider "virtualbox" do |v|
      v.memory = 16384
      v.cpus = 2
    end
    
    # Specify your hostname if you like
    # config.vm.hostname = "name"
    config.vm.box = "bento/ubuntu-20.04"
    config.vm.network "private_network", type: "dhcp"
    config.vm.provision "docker"
    # Specify the shared folder mounted from the host if you like
    # By default you get "." synced as "/vagrant"
    # config.vm.synced_folder ".", "/folder"  
    config.vm.provision "shell", inline: $script
  end

  $install_common_tools = <<-SCRIPT
  # bridged traffic to iptables is enabled for kube-router.
  cat >> /etc/ufw/sysctl.conf <<EOF
  net/bridge/bridge-nf-call-ip6tables = 1
  net/bridge/bridge-nf-call-iptables = 1
  net/bridge/bridge-nf-call-arptables = 1
  EOF
  
  # disable swap
  swapoff -a
  sed -i '/swap/d' /etc/fstab
  
  # Install kubeadm, kubectl and kubelet
  export DEBIAN_FRONTEND=noninteractive
  apt-get -qq install ebtables ethtool
  apt-get -qq update
  apt-get -qq install -y docker.io apt-transport-https curl
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
  deb http://apt.kubernetes.io/ kubernetes-xenial main
  EOF
  apt-get -qq update
  apt-get -qq install -y kubelet kubeadm kubectl
  SCRIPT
  
  $provision_master_node = <<-SHELL
  OUTPUT_FILE=/vagrant/join.sh
  rm -rf $OUTPUT_FILE
  
  # Start cluster
  sudo kubeadm init --apiserver-advertise-address=10.0.0.10 --pod-network-cidr=10.244.0.0/16 | grep "kubeadm join" > ${OUTPUT_FILE}
  chmod +x $OUTPUT_FILE
  
  # Configure kubectl
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
  # Fix kubelet IP
  echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=10.0.0.10"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
  
  # Configure flannel
  curl -o kube-flannel.yml https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
  sed -i.bak 's|"/opt/bin/flanneld",|"/opt/bin/flanneld", "--iface=enp0s8",|' kube-flannel.yml
  kubectl create -f kube-flannel.yml
  
  sudo systemctl daemon-reload
  sudo systemctl restart kubelet
  SHELL
  
  $install_multicast = <<-SHELL
  apt-get -qq install -y avahi-daemon libnss-mdns
  SHELL