server_ip = "192.168.21.10"
agents = { "agent1" => "192.168.21.11",
           "agent2" => "192.168.21.12" }

server_script = <<-SHELL
    sudo -i
    apk add curl
    curl -sfL https://get.k3s.io | sh -s - server --cluster-init --bind-address=#{server_ip} --node-external-ip=#{server_ip} --flannel-iface=eth1
    sleep 5
    cp /var/lib/rancher/k3s/server/token /vagrant_shared
    SHELL

agent_script = <<-SHELL
    sudo -i
    apk add curl
    curl -sfL https://get.k3s.io | K3S_TOKEN_FILE=/vagrant_shared/token K3S_URL=https://#{server_ip}:6443 INSTALL_K3S_EXEC="--flannel-iface=eth1" sh - 
    SHELL

Vagrant.configure("2") do |config|
  config.vm.box = "generic/alpine318"
  config.vm.synced_folder "./shared", "/vagrant_shared", create: true
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = 2
  end

  config.vm.define "server" do |server|
    server.vm.network "private_network", ip: server_ip
    server.vm.hostname = "server"
    server.vm.provision "shell", inline: server_script
  end

  agents.each do |name, ip|
    config.vm.define name do |agent|
      agent.vm.network "private_network", ip: ip
      agent.vm.hostname = name
      agent.vm.provision "shell", inline: agent_script
    end
  end
end