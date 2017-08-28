#
# NOTE: To resize disk run 'vagrant plugin install vagrant-disksize' from  https://github.com/sprotheroe/vagrant-disksize
# and uncomment the line  "nodeconfig.disksize.size = node[:disk]"
nodes = [
  {:hostname => 'icp-worker1', :ip => '192.168.122.11', :box => 'ubuntu/xenial64', :cpu => 2, :memory => 2048, :disk => '50GB'},
  {:hostname => 'icp-worker2', :ip => '192.168.122.12', :box => 'ubuntu/xenial64', :cpu => 2, :memory => 2048, :disk => '30GB'},
  # Here, here, here, add more worker nodes here.
  {:hostname => 'icp-master', :ip => '192.168.122.10', :box => 'ubuntu/xenial64', :cpu => 4, :memory => 6096, :disk => '30GB' },
]

icp_hosts = "[master]\n#{nodes.last[:ip]}\n[proxy]\n#{nodes.last[:ip]}\n[worker]\n"
vagrant_hosts = "127.0.0.1 localhost\n"
nodes.each do |node|
  icp_hosts = icp_hosts + node[:ip] + "\n" unless node == nodes.last
  vagrant_hosts = vagrant_hosts + "#{node[:ip]} #{node[:hostname]}\n"
end

# Please update the icp_config according to your laptop network
icp_config = '
network_type: calico
network_cidr: 10.1.0.0/16
service_cluster_ip_range: 10.0.0.0/24
ingress_enabled: true
mesos_enabled: false
install_docker_py: true
logs_max_age: 1
'

nfs_exports = '
/var/nfs/share1  *(rw,sync,no_root_squash,no_subtree_check)
/var/nfs/share2  *(rw,sync,no_root_squash,no_subtree_check)
/var/nfs/share3  *(rw,sync,no_root_squash,no_subtree_check)
/var/nfs/share4  *(rw,sync,no_root_squash,no_subtree_check)
/var/nfs/share5  *(rw,sync,no_root_squash,no_subtree_check)
/var/nfs/share6  *(rw,sync,no_root_squash,no_subtree_check)
'

# Please update if you want to use a specified version
icp_version = 'latest'

icp_hosts = "[master]\n#{nodes.last[:ip]}\n[proxy]\n#{nodes.last[:ip]}\n[worker]\n"
vagrant_hosts = "127.0.0.1 localhost\n"
nodes.each do |node|
  icp_hosts = icp_hosts + node[:ip] + "\n" unless (node == nodes.last && nodes.length != 1)
  vagrant_hosts = vagrant_hosts + "#{node[:ip]} #{node[:hostname]}\n"
end

Vagrant.configure(2) do |config|

  unless File.exists?('ssh_key')
    require "net/ssh"
    rsa_key = OpenSSL::PKey::RSA.new(2048)
    File.write('ssh_key', rsa_key.to_s)
    File.write('ssh_key.pub', "ssh-rsa #{[rsa_key.to_blob].pack("m0")}")
    File.chmod(0400, 'ssh_key')
  end

  rsa_public_key = IO.read('ssh_key.pub')
  rsa_private_key = IO.read('ssh_key')
  File.chmod(0400, 'ssh_key')

  config.vm.synced_folder ".", "/vagrant", disabled: false
  config.vm.provision "shell", inline: <<-SHELL
    mkdir -p /root/.ssh
    echo "#{rsa_public_key}" > /root/.ssh/authorized_keys
    passwd -u root
    echo root:Letmein123 | chpasswd
    sed -i 's/^PermitRootLogin.*/PermitRootLogin yes/g' /etc/ssh/sshd_config
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/g' /etc/ssh/sshd_config
    service sshd reload
    sudo apt-get update
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    sudo apt-get update
    apt-cache policy docker-ce
    sudo apt-get install -y docker-ce python-setuptools python-pip wget ntpdate 
    sudo systemctl status docker
    sudo sysctl -w net.ipv4.ip_forward=1
    sudo sysctl -w vm.max_map_count=262144
    pip install 'docker-py>=1.7.0'
    echo "#{vagrant_hosts}" > /etc/hosts
  SHELL

  nodes.each do |node|
    config.vm.define node[:hostname] do |nodeconfig|
      nodeconfig.vm.hostname = node[:hostname]
      nodeconfig.vm.box = node[:box]
      # Uncomment this in order to set the disk size
      #nodeconfig.disksize.size = node[:disk]
      nodeconfig.vm.box_check_update = false
      nodeconfig.vm.network "private_network", ip: node[:ip]
      nodeconfig.vm.provider "virtualbox" do |virtualbox|
        virtualbox.gui = false
        virtualbox.cpus = node[:cpu]
        virtualbox.memory = node[:memory]
      end

      if node == nodes.last
        nodeconfig.vm.provision "shell", inline: <<-SHELL
          #Set up NFS 
          sudo apt-get install -y nfs-kernel-server 
          mkdir -p /var/nfs/share1
          mkdir -p /var/nfs/share2
          mkdir -p /var/nfs/share3
          mkdir -p /var/nfs/share4
          mkdir -p /var/nfs/share5
          mkdir -p /var/nfs/share6
          echo "#{nfs_exports}" >> /etc/exports

          mkdir -p cluster
          echo "#{rsa_private_key}" > cluster/ssh_key
          chmod 400 cluster/ssh_key
          echo "#{icp_hosts}" > cluster/hosts
          echo "#{icp_config}" > cluster/config.yaml
          docker run -e LICENSE=accept -v "$(pwd)/cluster":/installer/cluster ibmcom/cfc-installer:"#{icp_version}" install
        SHELL
      end
    end
  end

end

