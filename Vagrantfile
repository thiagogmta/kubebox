# Definição da imagem linux base:
IMAGE_NAME = "ubuntu/focal64"
MASTER_SSH_FORWARDED_PORT = 2722

# Quantidade de worker nodes (caso precise de mais workers altere a quantidade)
N = 3

# Definições gerais de memória e processamento para cada Node
Vagrant.configure("2") do |config|
    config.vm.box = IMAGE_NAME
    config.vm.box_check_update = false
    config.vm.network :forwarded_port, guest: 22, host: 2222, id: "ssh", disabled: true

    # Provisionamento Master Node
    config.vm.define "master" do |master|
        master.vm.provider "virtualbox" do |v|
            v.memory = 2048
            v.cpus = 2
            v.name = "master"
        end

        master.vm.hostname = "master"
        master.vm.network "private_network", ip: "192.168.50.10"
        master.vm.network "forwarded_port", guest: 22, host: MASTER_SSH_FORWARDED_PORT, auto_correct: true
        master.vm.network "forwarded_port", guest: 6443, host: 6443, auto_correct: true
    end


    # Provisionamento dos Worker Nodes
    (1..N).each do |i|
        node_ip = "192.168.50.#{i + 10}" # Define o IP para cada worker node
        hostname = "worker-#{'%02d' % i}" # Define o hostname com formato padronizado
        forwarded_port = MASTER_SSH_FORWARDED_PORT + i + 1 # Define a porta SSH para o redirecionamento
        
        config.vm.define hostname do |node|
            node.vm.provider "virtualbox" do |v|
                v.memory = 2048
                v.cpus = 2
                v.name = "#{hostname}"
            end
            node.vm.hostname = "#{hostname}"
            node.vm.network "private_network", ip: node_ip
            node.vm.network "forwarded_port", guest: 22, host: forwarded_port, auto_correct: true
        end
    end
end