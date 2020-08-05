Vagrant.configure("2") do |config|
  config.vm.box = "genebean/centos-7-nocm"

  # this controls how many nodes are created
  node_num=3

  (1..node_num).each do |i|
    if i == 1 then
      vm_name="master"
    else
      vm_name="node#{i-1}"
    end

    config.vm.define vm_name do |s|
      s.vm.hostname=vm_name
      s.vm.network "private_network", ip: "192.168.50.#{i+10}"
      s.vm.provision "shell", inline: 'echo "GATEWAYDEV=eth0" >> /etc/sysconfig/network && systemctl restart network'
      s.vm.provision "shell", inline: 'yum install -y policycoreutils-python'
      if i == 1 then
        # For Master
        s.vm.provision "shell", inline:<<-SHELL
          set -e

          # these two are needed to install k3s
          yum install -y container-selinux selinux-policy-base https://rpm.rancher.io/k3s-selinux-0.1.1-rc1.el7.noarch.rpm

          # Store private network IP address in variable
          IPADDR=$(ip a show eth1 | grep "inet " | awk '{print $2}' | cut -d / -f1)
        
          # Deploy k3s, specify private IP as kubelet's IP, disable loadbalancer and traefik
          curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--no-deploy=servicelb --no-deploy=traefik --node-ip=${IPADDR} --flannel-iface=eth1 --write-kubeconfig-mode 644" sh -
        
          # Place token for agent registration in shared folder
          cp /var/lib/rancher/k3s/server/node-token /vagrant/token
        
          /usr/local/bin/kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
          /usr/local/bin/kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
          /usr/local/bin/kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
          /usr/local/bin/kubectl apply -f /vagrant/metallb-configmap.yml
        
          # make sure the kube config is found
          echo "export KUBECONFIG='/etc/rancher/k3s/k3s.yaml'" > /etc/profile.d/k3s-kubeconfig.sh
          source /etc/profile.d/k3s-kubeconfig.sh
        
          # install helm
          export PATH=/usr/local/bin:$PATH
          curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
        
          # OpenFaaS uses the shasum program
          yum install -y perl-Digest-SHA
        SHELL
      else
        # For Nodes
        s.vm.provision "shell", inline:<<-SHELL
          set -e

          # these two are needed to install k3s
          yum install -y container-selinux selinux-policy-base https://rpm.rancher.io/k3s-selinux-0.1.1-rc1.el7.noarch.rpm

          export K3S_TOKEN=$(cat /vagrant/token)
          export K3S_URL=https://192.168.50.11:6443
        
          # Store private network IP address in variable
          IPADDR=$(ip a show eth1 | grep "inet " | awk '{print $2}' | cut -d / -f1)
        
          curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--node-ip=${IPADDR} --flannel-iface=eth1" sh -
        SHELL
      end
    end
  end
end

