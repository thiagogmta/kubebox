# Kubebox - kubernetes whit istio on virtualbox automated by vagrant and provisioned by ansible

Esse repositório apresenta o Kubebox uma automação de ambiente de cluster Kubernetes com Kubeadm, Containerd e istio service mesh automatizado por Vagrant e Provisionado via Ansible.

Thiago Guimarães Tavares   
thiagogmta@ifto.edu.br

## Ambiente e Requisitos

Este projeto foi executado em um Notebook intel core i5 10th gen com 24GB de ram, 128gb de armazenamento SSD em um sistema operacional Linux distribuição Pop!_OS 22.04 LTS. O ambiente a ser criado requer 8gb de memória RAM.

As ferramentas a seguir devem estar instaladas no ambiente:

- [Vagrant](https://developer.hashicorp.com/vagrant/downloads)
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html)
- [Oracle VirtualBox](https://www.virtualbox.org/wiki/Downloads)

> Até o momento, não foram realizados testes em ambientes Windows com WSL2 e VirtualBox. Embora exista a possibilidade de sucesso na execução, não é possível garantir que funcione conforme esperado.

## Arquivos do Repositório

- inventory/vagrant.hosts - Define quais máquinas serão provisionadas
- playbooks/files/config.toml - Arquivo de configuração do ContainerD para o ambiente
- playbooks/files/joincluster - Vincula os worker ao master node
- playbooks/templates/calico-custom-resource.yaml.j2 - Configura o calico como provedor de rede
- Vagrantfile - Define as características dos nós do cluster

## Vagrant e Ansible

**Vagrant** é uma ferramenta que automatiza a criação de máquinas virtuais. O vagrant irá criar quatro VM's com ubuntu server sendo:
- k8s-master - Este será o Master Node do Cluster
- k8s-worker-01 - Este será o Worker Node 1 do Cluster
- k8s-worker-02 - Este será o Worker Node 2 do Cluster
- k8s-worker-03 - Este será o Worker Node 3 do Cluster

**Ansible** é uma ferramenta de automação de infraestrutura. Que irá provisionar o cluster com as seguintes características:

- Ubuntu Focal64 20.04
- ContainerD
- Kubeadm
- Kubelet
- Kubectl
- Helm

## Criando o Ambiente

Clone este repositório e execute o Vagrantfile:

```bash
git clone https://github.com/thiagogmta/kubebox.git
cd kubebox
vagrant up
```

![Vagrant Up](/img/kubebox.png)

Podemos verificar o funcionamento das VM's através da interface gráfica do Virtualbox ou pela CLI do vagrant:

```bash
vagrant status
```

## Provisionando o Ambiente

Após todas as VM's terem sido iniciadas faremos o provisionamento com Ansible.

```bash
ansible-playbook -i inventory/vagrant.hosts playbooks/ansible-playbook.yaml
```

O playbook irá preparar o ambiente com os pacotes necessários.

![Vagrant Up](/img/ansible.png)

Ao final do processo podemos acessar o Master node e verificar o funcionamento do cluster.

```bash
vagrant ssh k8s-master
kubectl get nodes
```

![Vagrant Up](/img/nodes.png)

> Observação: Os nodes podem demorar alguns minutos para que seu status seja apresentado como "Ready".

Enjoy! Seu cluster está pronto para receber aplicações de teste.

## Instalação do Istio Service Mesh

No Master Node iremos executar os comandos necessários:

1. Download do binário de instalação

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio*
export PATH=$PWD/bin:$PATH
```

2. Instalação do istio

```bash
istioctl install --set profile=demo
kubectl label namespace default istio-injection=enabled
```

![Instalação istio](/img/istio.png)

3. Ativação da injeção automática do sidecar em todos os pods do namespace.

```bash
kubectl label namespace default istio-injection=enabled
```

4. Verificando a instalação do Istio

    - Lista todos os serviços no  namespace: istio-system
    - Lista todos os pods em execução no namespace: istio-system
    - Exibe todos os recursos (pods, serviços, deployments, etc)

```bash
kubectl get svc -n istio-system
kubectl get pods -n istio-system
kubectl get all
```

## Pacote de Serviços Complementares

Nesta sessão é apresentado a instalação e configuração de ferramentas complementares ao cluster:

- Kugernetes Dashboard
- Prometheus
- Jaeger
- Grafana
- Kiali

### Kubernetes Dashboard

É uma interface web de usuário que permite a visualização e o gerenciamento do cluster.

Instalação:

```bash
# Adicionando o repositório do kubernetes-dashboard
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
# Deploy do dashboard usando Helm
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
```

## Alterando a quantidade de Nós ou Recursos exigidos

Para alterar a quantidade de recursos exigidos pelos Nós basta alterar os parâmetros a seguir no arquivo Vagrantfile:

```bash
v.memory = 2048
v.cpus = 2
```

Para alterar a quantidade de Worker Nodes basta:
1. Alterar a quantidade na variável "N" do arquivo Vagrantfile
2. Inserir (ou remover) a entrada no arquivo /inventory/vagrant.hosts de acordo com a quantidade de nós estipulada em "N". Respeite a sitaxe e sequência do endereçamento IP.

## Referência

Esse repoistório é baseia-se nos repositório de [Alvinditya Saputra](https://github.com/piinalpin/home-lab-provisioning) e [Javier Ruiz Jiménez](https://github.com/itwonderlab/ansible-vbox-vagrant-kubernetes/tree/master).