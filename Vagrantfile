# -*- mode: ruby -*-
# vi: set ft=ruby :

# Workerノードの数
worker_count=2

# 共通のプロビジョニングスクリプト
$configureBox = <<-SHELL

  # パッケージ更新
  yum update -y

  # Dockerの前提パッケージ
  yum install -y yum-utils device-mapper-persistent-data lvm2
  # Dockerのレポジトリ追加
  yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  # Dockerのインストール
  # VERSION=$(yum list docker-ce --showduplicates | sort -r | grep 17.03 | head -1 | awk '{print $2}')
  # yum install -y --setopt=obsoletes=0 docker-ce-$VERSION docker-ce-selinux-$VERSION
  VERSION=$(yum list docker-ce --showduplicates | sort -r | grep 18.06 | head -1 | awk '{print $2}')
  yum install -y docker-ce-$VERSION

  # Dockerデーモンの設定
  mkdir -p /etc/docker
  cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
  mkdir -p /etc/systemd/system/docker.service.d
  systemctl enable docker
  systemctl daemon-reload
  systemctl restart docker

  # vagrantユーザーをdockerグループに追加
  usermod -aG docker vagrant

  # スワップを無効化する
  swapoff -a
  # プロビジョニングで実行する場合はバックスラッシュのエスケープが必要なことに注意
  # sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
  sed -i '/ swap / s/^\\(.*\\)$/#\\1/g' /etc/fstab

  # Kubernetesのレポジトリ追加
  cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF
  
  # SELinuxを無効化
  setenforce 0
  sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
  
  # kubeadm、kubelet、kubectlのインストール
  VERSION=$(yum list kubeadm --showduplicates --disableexcludes=kubernetes | sort -r | grep 1.14 | head -1 | awk '{print $2}')
  yum install -y kubeadm-$VERSION kubelet-$VERSION kubectl-$VERSION --disableexcludes=kubernetes
  # yum install -y kubeadm kubelet kubectl --disableexcludes=kubernetes
  systemctl enable kubelet && systemctl start kubelet

  cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
  sysctl --system

  # プライベートネットワークのNICのIPアドレスを変数に格納
  IPADDR=$(ip a show eth1 | grep inet | grep -v inet6 | awk '{print $2}' | cut -f1 -d/)
  # kubeletがプライベートネットワークのNICにバインドするように設定
  sed -i "/KUBELET_EXTRA_ARGS=/c\KUBELET_EXTRA_ARGS=--node-ip=$IPADDR" /etc/sysconfig/kubelet
  # kubeletを再起動
  systemctl daemon-reload
  systemctl restart kubelet

SHELL

# Masterノードのプロビジョニングスクリプト
$configureMaster = <<-SHELL

  echo "This is master"

  # プライベートネットワークのNICのIPアドレスを変数に格納
  IPADDR=$(ip a show eth1 | grep inet | grep -v inet6 | awk '{print $2}' | cut -f1 -d/)
  # ホスト名を変数に格納
  HOSTNAME=$(hostname -s)

  # kubeadm initの実行
  # Flannel
  # kubeadm init --apiserver-advertise-address=$IPADDR --apiserver-cert-extra-sans=$IPADDR --node-name $HOSTNAME --pod-network-cidr=10.244.0.0/16
  # Calico
  kubeadm init --apiserver-advertise-address=$IPADDR --apiserver-cert-extra-sans=$IPADDR --node-name $HOSTNAME --pod-network-cidr=192.168.0.0/16

  # vagrantユーザーがkubectlを実行できるようにする
  sudo --user=vagrant mkdir -p /home/vagrant/.kube
  cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
  chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

  # Flannelのインストール
  # export KUBECONFIG=/etc/kubernetes/admin.conf
  # kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
  # Calicoのインストール
  export KUBECONFIG=/etc/kubernetes/admin.conf
  kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml
  kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml

  # kubectl joinコマンドを保存する
  kubeadm token create --print-join-command > /etc/kubeadm_join_cmd.sh
  chmod +x /etc/kubeadm_join_cmd.sh

  # sshでのパスワード認証を許可する
  sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
  systemctl restart sshd

  # jq
  VERSION="1.6"
  curl -Lo jq https://github.com/stedolan/jq/releases/download/jq-${VERSION}/jq-linux64
  chmod +x jq
  mv jq /usr/local/bin/

  # helm
  VERSION="v2.13.1"
  curl -LO https://storage.googleapis.com/kubernetes-helm/helm-${VERSION}-linux-amd64.tar.gz
  tar zxvf helm-${VERSION}-linux-amd64.tar.gz
  cp linux-amd64/helm /usr/local/bin/
  rm -rf linux-amd64
  rm -f helm-${VERSION}-linux-amd64.tar.gz

  # git
  yum -y install git

  # kubens/kubectx
  git clone https://github.com/ahmetb/kubectx.git /opt/kubectx
  ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
  ln -s /opt/kubectx/kubens /usr/local/bin/kubens

  # kube-ps1
  git clone https://github.com/jonmosco/kube-ps1.git /opt/kube-ps1
  cat <<'EOF' >> /home/vagrant/.bashrc
LANG=en_US.UTF-8
source /opt/kube-ps1/kube-ps1.sh
KUBE_PS1_SUFFIX=') '
PS1='$(kube_ps1)'$PS1
EOF

  # kubectlの補完を有効にする
  echo "source <(kubectl completion bash)" >> /home/vagrant/.bashrc

SHELL

# Workerノードのプロビジョニングスクリプト
$configureNode = <<-SHELL

  echo "This is worker"

  yum install -y sshpass
  # sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@192.168.33.11:/etc/kubeadm_join_cmd.sh .
  sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@172.16.33.11:/etc/kubeadm_join_cmd.sh .
  sh ./kubeadm_join_cmd.sh

SHELL

Vagrant.configure(2) do |config|

  (1..worker_count+1).each do |i|

    if i == 1 then
      vm_name = "master"
    else
      vm_name = "node#{i-1}"
    end
      
    config.vm.define vm_name do |s|

      # ホスト名
      s.vm.hostname = vm_name
      # ノードのベースOSを指定
      s.vm.box = "centos/7"
      # ネットワークを指定
      # pod-network-cidrと重ならないように注意
      # private_ip = "192.168.33.#{i+10}"
      private_ip = "172.16.33.#{i+10}"
      s.vm.network "private_network", ip: private_ip

      # ノードのスペックを指定
      s.vm.provider "virtualbox" do |v|
        v.gui = false        
        if i == 1 then
          v.cpus = 2
          v.memory = 1024
        else
          v.cpus = 1
          v.memory = 1024
        end
      end

      # 共通のプロビジョニング
      s.vm.provision "shell", inline: $configureBox

      if i == 1 then
        # Masterのプロビジョニング
        s.vm.provision "shell", inline: $configureMaster
      else
        # Nodeのプロビジョニング
        s.vm.provision "shell", inline: $configureNode
      end
    
    end
  end
end
