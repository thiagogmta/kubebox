
# Kubebox - Kubernetes no VirtualBox automatizado por Vagrant e provisionado por Ansible

Este repositório apresenta o **Kubebox**, uma automação de ambiente de cluster Kubernetes com Kubeadm e Containerd automatizado por Vagrant e provisionado via Ansible.

Thiago Guimarães Tavares  
[thiagogmta@ifto.edu.br](mailto:thiagogmta@ifto.edu.br)

## Ambiente e Requisitos

Este projeto foi executado em um notebook com processador Intel Core i5 10ª geração, 24GB de RAM e 128GB de armazenamento SSD, utilizando o sistema operacional Linux Pop!_OS 22.04 LTS. O ambiente a ser criado requer, no mínimo, 8GB de memória RAM.

As ferramentas [Oracle VirtualBox](https://www.virtualbox.org/wiki/Downloads), [Vagrant](https://developer.hashicorp.com/vagrant/downloads) e [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html) devem estar instaladas no ambiente. Para nossos testes foram utilizadas as referidas ferramentas nas versões:

- Oracle VirtualBox - 6.1
- Vagrant - 2.4.1
- Ansible - 2.16.10

Ao final desta implantação, o cluster contará com as seguintes ferramentas:

- **Kubernetes Dashboard**  
  Interface gráfica para gerenciar e monitorar clusters Kubernetes.
<!-- - **Istio Service Mesh**  
  Malha de serviços que facilita a comunicação entre microserviços, fornecendo balanceamento de carga, observabilidade, segurança e roteamento de tráfego. -->
- **Prometheus**  
  Sistema de monitoramento que coleta e armazena métricas em tempo real, permitindo o monitoramento do desempenho de aplicações.
- **Grafana**  
  Plataforma de visualização de dados que permite criar dashboards interativos e gráficos a partir de várias fontes de dados, como o Prometheus.
<!--
- **Jaeger**  
  Ferramenta de rastreamento distribuído utilizada para monitorar e solucionar problemas de desempenho em sistemas de microserviços, identificando gargalos em transações complexas.

- **Kiali**  
  Interface de gerenciamento e observabilidade para o Istio Service Mesh, facilitando a visualização de topologias de serviços, monitoramento de tráfego e diagnósticos de problemas em ambientes de microserviços.
-->

> Observação: Até o momento, não foram realizados testes em ambientes Windows com WSL2 e VirtualBox. Embora seja possível que funcione, não há garantias.

## Arquivos do Repositório

- `inventory/vagrant.hosts` - Define quais máquinas serão provisionadas.
- `playbooks/files/config.toml` - Arquivo de configuração do Containerd para o ambiente.
- `playbooks/files/joincluster` - Comando para vincular os nós workers ao nó master.
- `playbooks/templates/calico-custom-resource.yaml.j2` - Configura o Calico como provedor de rede.
- `Vagrantfile` - Define as características dos nós do cluster.

## Vagrant e Ansible

**Vagrant** é uma ferramenta que automatiza a criação de máquinas virtuais. O Vagrant criará quatro VMs com Ubuntu Server, sendo:
- `k8s-master` - O nó master do cluster.
- `k8s-worker-01` - O nó worker 1 do cluster.
- `k8s-worker-02` - O nó worker 2 do cluster.
- `k8s-worker-03` - O nó worker 3 do cluster.

**Ansible** é uma ferramenta de automação de infraestrutura que provisionará o cluster com as seguintes características:

- Ubuntu Focal64 20.04
- ContainerD
- Kubeadm
- Kubelet
- Kubectl
- Helm

## Criando o Ambiente

Clone este repositório e execute o Vagrant para criar as máquinas virtuais:

```bash
git clone https://github.com/thiagogmta/kubebox.git
cd kubebox
vagrant up
```

Podemos verificar o funcionamento das VMs pela interface gráfica do VirtualBox ou pela CLI do Vagrant:

```bash
vagrant status
```

![Vagrant Up](/img/kubebox.png)

### Provisionando o Ambiente

Após todas as VMs terem sido iniciadas, faremos o provisionamento com Ansible:

```bash
ansible-playbook -i inventory/vagrant.hosts playbooks/ansible-playbook.yaml
```

O playbook instalará os pacotes necessários no ambiente.

![Ansible Playbook](/img/ansible.png)

Ao final do processo, podemos acessar o nó master e verificar o funcionamento do cluster:

```bash
vagrant ssh master
kubectl get nodes
```

![Get Nodes](/img/nodes.png)

> Observação: Pode levar alguns minutos para que os nós sejam apresentados como "Ready".

Verificando o endereço IP do nó master:

```bash
kubectl cluster-info

# Saída do comando:
Kubernetes control plane is running at https://192.168.50.10:6443
CoreDNS is running at https://192.168.50.10:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

Seu cluster está pronto para receber aplicações de teste.

<!-- ## Istio Service Mesh

Nesta seção, faremos a instalação do Istio Service Mesh no cluster:

```bash
# Criando o namespace istio-system que abrigará os serviços (Istio, Prometheus, Grafana e Kiali)
kubectl create namespace istio-system

# Adicionando o repositório
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

# Instalando o pacote base do Istio
helm install istio-base istio/base -n istio-system --set defaultRevision=default

# Retorno:
NAME: istio-base
LAST DEPLOYED: Tue Sep 17 12:35:32 2024
NAMESPACE: istio-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Istio base successfully installed!
```

```bash
# Instalando o plano de controle do Istio
helm install istiod istio/istiod -n istio-system --wait

# Retorno:
NAME: istiod
LAST DEPLOYED: Wed Sep 18 14:11:56 2024
NAMESPACE: istio-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
"istiod" successfully installed!
```

Ativação da injeção automática do sidecar em todos os pods do namespace:

```bash
kubectl label namespace default istio-injection=enabled
```

Podemos consultar todos os pods, serviços e deployments do cluster com:

```bash
kubectl get all -A
``` -->

## Instalação e Configuração: Kubernetes Dashboard, Prometheus e Grafana

Nesta seção, será apresentada a instalação e configuração de ferramentas complementares ao cluster:

- Kubernetes Dashboard
- Prometheus
- Grafana

O fluxo seguirá conforme a seguir:
1. Criaremos um namespace para o Kubernetes Dashboard intitulado `kubernetes-dashboard` e um segundo namespace para o Prometheus e Grafana intitulado `monitoring`.
2. Adicionaremos os repositórios Helm dos serviços e atualizaremos os repositórios.
3. Instalaremos os serviços via Helm.
4. Para acessar os serviços fora do cluster, será necessário expor uma porta para o acesso externo. Criaremos um único arquivo YAML contendo as configurações para expor os serviços (Kubernetes Dashboard, Prometheus e Grafana) via NodePort.


### Adicionando os Repositórios

```bash
# Criando Namespaces
kubectl create namespace kubernetes-dashboard
kubectl create namespace monitoring

# Adicionando o repositório do Kubernetes Dashboard
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/

# Adicionando o repositório do Prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Adicionando o repositório do Grafana
helm repo add grafana https://grafana.github.io/helm-charts

# Atualizando os repositórios Helm
helm repo update
```

### Instalação

```bash
# Instalando o Kubernetes Dashboard
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard -n kubernetes-dashboard

# Instalando o Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring

# Instalando o Grafana
helm install my-grafana grafana/grafana -n monitoring
```

Caso seja necessário desinstalar os serviços:

```bash
# Desinstalar o Kubernetes Dashboard
helm uninstall kubernetes-dashboard -n kubernetes-dashboard

# Desinstalar o Prometheus
helm uninstall prometheus -n monitoring

# Desinstalar o Grafana
helm uninstall my-grafana -n monitoring

# Remover os namespaces
kubectl delete namespace kubernetes-dashboard
kubectl delete namespace monitoring
```

### Verificando os Serviços

```bash
# Lista todos os recursos de todos os namespaces
kubectl get all -A

# Lista todos os serviços do namespace kubernetes-dashboard
kubectl get svc -n kubernetes-dashboard

# Lista todos os serviços do namespace istio-system
kubectl get svc -n monitoring
```

### Expondo os Serviços via NodePort

Os serviços estão acessíveis via **ClusterIP**, que é o tipo de serviço padrão do Kubernetes, e só estão disponíveis internamente no cluster. Para expor os serviços externamente, precisamos configurar o **NodePort**. Criaremos um arquivo chamado `services-nodeport.yaml` com o seguinte conteúdo:

```bash
nano services-nodeport.yaml
```

```yaml
# Kubernetes Dashboard
kind: Service
apiVersion: v1
metadata:
  labels:
    app: kong
    release: kubernetes-dashboard
  name: kubernetes-dashboard-kong-proxy 
  namespace: kubernetes-dashboard
spec:  
  type: NodePort
  ports:
    - nodePort: 32364
      port: 443
      protocol: TCP
      targetPort: 8443
  selector:    
    app.kubernetes.io/name: kong
    app.kubernetes.io/instance: kubernetes-dashboard
---
# Prometheus
apiVersion: v1
kind: Service
metadata:
  labels:
    app: prometheus-operated
  name: prometheus-operated-np
  namespace: monitoring
spec:
  type: NodePort
  ports:
    - nodePort: 32365
      port: 9090
      protocol: TCP
      targetPort: 9090
  selector:
    app.kubernetes.io/name: prometheus
---
# Grafana
apiVersion: v1
kind: Service
metadata:
  labels:
    app: grafana
  name: my-grafana
  namespace: monitoring
spec:
  type: NodePort
  ports:
    - nodePort: 32366
      port: 80
      protocol: TCP
      targetPort: 3000
  selector:
    app.kubernetes.io/name: grafana
```

Aplique as configurações do arquivo com:

```bash
kubectl apply -f services-nodeport.yaml
```

Verificando o NodePort dos serviços:

```bash
kubectl get svc -n kubernetes-dashboard
kubectl get svc -n monitoring
```

```bash
# Saida do comando: Kubectl get svc -n kubernetes-dashboard
NAME                                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard-api               ClusterIP   10.96.8.40       <none>        8000/TCP        6m28s
kubernetes-dashboard-auth              ClusterIP   10.102.111.154   <none>        8000/TCP        6m28s
kubernetes-dashboard-kong-proxy        NodePort    10.109.154.128   <none>        443:32364/TCP   6m28s
kubernetes-dashboard-metrics-scraper   ClusterIP   10.98.7.254      <none>        8000/TCP        6m28s
kubernetes-dashboard-web               ClusterIP   10.101.79.40     <none>        8000/TCP        6m28s

# Saída do comando: kubectl get svc -n monitoring
NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   94m
my-grafana                                NodePort    10.97.254.93     <none>        80:32366/TCP                 100m
prometheus-grafana                        ClusterIP   10.102.93.162    <none>        80/TCP                       94m
prometheus-kube-prometheus-alertmanager   ClusterIP   10.109.8.238     <none>        9093/TCP,8080/TCP            94m
prometheus-kube-prometheus-operator       ClusterIP   10.108.214.142   <none>        443/TCP                      94m
prometheus-kube-prometheus-prometheus     ClusterIP   10.109.176.55    <none>        9090/TCP,8080/TCP            94m
prometheus-kube-state-metrics             ClusterIP   10.102.140.77    <none>        8080/TCP                     94m
prometheus-operated                       ClusterIP   None             <none>        9090/TCP                     94m
prometheus-operated-np                    NodePort    10.110.101.229   <none>        9090:32365/TCP               95m
prometheus-prometheus-node-exporter       ClusterIP   10.103.237.251   <none>        9100/TCP                     94m
```

Para acessar os serviços no navegador, utilize os endereços a seguir:

- **Kubernetes Dashboard**: https://192.168.50.10:32364
- **Prometheus**: http://192.168.50.10:32365
- **Grafana**: http://192.168.50.10:32366

## Acessando os Serviços: Kubernetes Dashboard, Prometheus, Grafana e Kiali

Nesta seção, são apresentados os detalhes de acesso a cada ferramenta.

### Kubernetes Dashboard

Ao acessar o Dashboard, será apresentada a mensagem de alerta "Potencial risco de segurança à frente". Selecione **Avançado** e depois **Aceitar o risco e continuar**. Para acessar o Kubernetes Dashboard, precisaremos de um usuário e token de acesso.

![Kubernetes Dashboard](/img/login-dashboard.png)

Vamos criar um arquivo de implantação `user-dashboard.yaml` que:

- Cria um usuário para acesso ao Dashboard.
- Atribui a função de administrador ao usuário.
- Cria um token para autenticação no Kubernetes Dashboard.

```yaml
nano user-dashboard.yaml
```

```yaml
--- 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
--- 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: v1
kind: Secret
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: "admin-user"
type: kubernetes.io/service-account-token
```

Aplicaremos com:

```bash
kubectl apply -f user-dashboard.yaml
```

Para verificar o token do `admin-user`:

```bash
kubectl -n kubernetes-dashboard describe secret admin-user
```

Copie o token e cole na janela de login para acessar o Kubernetes Dashboard.

![Kubernetes Dashboard](/img/dashboard.png)

### Prometheus

O Prometheus apresenta sua interface diretamente.

![Prometheus](/img/prometheus.png)

### Grafana

Para efetuar login no Grafana, utilize o usuário `admin`. Para obter a senha, utilize o comando:

```bash
kubectl get secret -n monitoring my-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Copie e cole a senha no campo "Password".

![Login Grafana](/img/grafana2.png)

**Habilitando Armazenamento Persistente (opicional)**

Por padrão o armazenamento persistente (persistent storage) do Grafana vem desativado, o que significa que os dados serão perdidos quando o contêiner for finalizado ou reiniciado. Para habilitar o armazenamento persistente crie no cluster um arquivo `values.yaml`. Copie o conteúdo do arquivo `pvc/values.yaml` deste repositório e cole no arquivo criado. Habilite o armazenamento persistente com:

```bash
helm upgrade grafana grafana/grafana -f values.yaml -n monitoring
```

## Alterando a Quantidade de Nós ou Recursos Exigidos

Para alterar a quantidade de recursos exigidos pelos nós, basta alterar os parâmetros a seguir no arquivo `Vagrantfile`:

```bash
v.memory = 2048
v.cpus = 2
```

Para alterar a quantidade de workers nodes:

1. Altere a quantidade na variável `N` no arquivo `Vagrantfile`.
2. Insira (ou remova) a entrada correspondente no arquivo `inventory/vagrant.hosts`, de acordo com a quantidade de nós estipulada em `N`, respeitando a sintaxe e sequência de endereços IP.

## Referências

Este repositório é baseado nos seguintes materiais:

- [Alvinditya Saputra](https://github.com/piinalpin/home-lab-provisioning)
- [Javier Ruiz Jiménez](https://github.com/itwonderlab/ansible-vbox-vagrant-kubernetes/tree/master)
- [Vladmir Romanov](https://www.kerno.io/learn/kubernetes-dashboard-deploy-visualize-cluster)