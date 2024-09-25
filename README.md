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

Ao final desta implantação o cluster irá contar com as seguintes ferramentas:

- Kubernetes Dashboard
  - Interface gráfica para gerenciar e monitorar clusters Kubernetes
- Istio Service Mesh
  - Malha de serviços que facilita a comunicação entre microserviços, fornecendo recursos como balanceamento de carga, observabilidade, segurança, e roteamento de tráfego.
- Prometheus
  - Sistema de monitoramento que coleta e armazena métricas em tempo real, permitindo o monitoramento de desempenho de aplicações.
- Grafana
  - Plataforma de visualização de dados que permite criar dashboards interativos e gráficos a partir de várias fontes de dados, como Prometheus.
<!--
- Jaeger
  - Ferramenta de rastreamento distribuído, utilizada para monitorar e solucionar problemas de desempenho em sistemas de microserviços, identificando gargalos em transações complexas.
-->

- Kiali
  - Interface de gerenciamento e observabilidade para Istio Service Mesh, que facilita a visualização de topologias de serviços, monitoramento de tráfego e diagnósticos de problemas em ambientes de microserviços.

> Até o momento, não foram realizados testes em ambientes Windows com WSL2 e VirtualBox. Embora exista a possibilidade de sucesso na execução, não é possível garantir que funcione conforme esperado.

## Arquivos do Repositório

- inventory/vagrant.hosts - Define quais máquinas serão provisionadas
- playbooks/files/config.toml - Arquivo de configuração do ContainerD para o ambiente
- playbooks/files/joincluster - Vincula os workers ao master node
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

Clone este repositório e execute o Vagrantfile para criação das máquinas virtuais:

```bash
git clone https://github.com/thiagogmta/kubebox.git
cd kubebox
vagrant up
```

Podemos verificar o funcionamento das VM's através da interface gráfica do Virtualbox ou pela CLI do vagrant:

```bash
vagrant status
```

![Vagrant Up](/img/kubebox.png)

### Provisionando o Ambiente

Após todas as VM's terem sido iniciadas faremos o provisionamento com Ansible.

```bash
ansible-playbook -i inventory/vagrant.hosts playbooks/ansible-playbook.yaml
```

O playbook irá preparar o ambiente com os pacotes necessários.

![Vagrant Up](/img/ansible.png)

Ao final do processo podemos acessar o Master node e verificar o funcionamento do cluster.

```bash
vagrant ssh master
kubectl get nodes
```

![Vagrant Up](/img/nodes.png)

> Observação: Os nodes podem demorar alguns minutos para que seu status seja apresentado como "Ready".

Verificando o endereço IP do Master Node:

```bash
kubectl cluster-info

# Saida do comando:
Kubernetes control plane is running at https://192.168.50.10:6443
CoreDNS is running at https://192.168.50.10:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

Enjoy! Seu cluster está pronto para receber aplicações de teste.

## Istio Service Mesh

Nesta sessão faremos a instalação do Istio Service Mesh no cluster

```bash
# Criando o Namespace Istio-system que irá abrigar os serviços (Istio, Prometheus, Grafana e Kiali)
kubectl create namespace istio-system

# Adicionando o repositório
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

# Instalar o pacote base do Istio
helm install istio-base istio/base -n istio-system --set defaultRevision=default

# Retorno
NAME: istio-base
LAST DEPLOYED: Tue Sep 17 12:35:32 2024
NAMESPACE: istio-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Istio base successfully installed!

To learn more about the release, try:
  $ helm status istio-base -n istio-system
  $ helm get all istio-base -n istio-system
```

```bash
# Instalar o plano de controle do Istio
helm install istiod istio/istiod -n istio-system --wait

# Retorno
NAME: istiod
LAST DEPLOYED: Wed Sep 18 14:11:56 2024
NAMESPACE: istio-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
"istiod" successfully installed!

To learn more about the release, try:
  $ helm status istiod -n istio-system
  $ helm get all istiod -n istio-system
```

Ativação da injeção automática do sidecar em todos os pods do namespace.

```bash
kubectl label namespace default istio-injection=enabled
```

Podemos consultar todos os pods, services e deployments do cluster com:

```bash
kubectl get all -A
```

## Instalação e Configuração: Kubernetes Dashboard, Prometheus, Grafana e Kiali

Nesta sessão é apresentado a instalação e configuração de ferramentas complementares ao cluster:

- Kugernetes Dashboard
- Istio Service Mesh
- Prometheus
- Grafana
- Kiali

Para isso adotaremos a seguinte estratégia:
1. Criaremos um namespace para o Kubernetes Dashboard. Os serviços Prometheus, Grafana e Kiali ficarão no namespace istio-system
2. Adicionaremos os repositórios helm dos serviços e posteriormente atualizaremos o repositório
3. Realizaremos a instalação dos serviços via helm
4. Para acessar os serviços fora do cluster será necessário expor uma porta que permita esse acesso externo. Criaremos um único arquivo yaml contendo as configurações necessárias para expor os serviços (Prometheus, Grafana e Kiali) para que sejam acessíveis fora do cluster através da NodePort.

### Adicionando os Repositórios

```bash
# Criando o Namespace par ao Dashboard
kubectl create namespace kubernetes-dashboard

# Adicionar o repositório do Kubernetes Dashboard
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/

# Adicionando o repositório do Prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Adicionando repositório do Grafana
helm repo add grafana https://grafana.github.io/helm-charts

# Adicionando Repositório do Kiali
helm repo add kiali https://kiali.org/helm-charts

# Atualizando o Repositório Helm
helm repo update

# Retorno
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "kubernetes-dashboard" chart repository
...Successfully got an update from the "kiali" chart repository
...Successfully got an update from the "istio" chart repository
...Successfully got an update from the "grafana" chart repository
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈
```

### Instalação

```bash
# Instalando o Dashboard
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard -n kubernetes-dashboard

# Instalação do Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack -n istio-system

# Instalação do Grafana
helm install grafana grafana/grafana -n istio-system

# Instalando do Kiali Server
helm install kiali-server kiali/kiali-server -n istio-system
```

Caso seja necessário desinstalar os serviços:

```bash
# Desinstalar o Kubernetes Dashboard
helm uninstall kubernetes-dashboard -n kubernetes-dashboard

# Desinstalar o Prometheus
helm uninstall prometheus -n istio-system

# Desinstalar o Grafana
helm uninstall grafana -n istio-system

# Desinstalar o Kiali
helm uninstall kiali-server -n istio-system

# Remover os namespaces
kubectl delete namespace kubernetes-dashboard
kubectl delete namespace istio-system
```

Para verificar os serviços:

```bash
# Lista todos os recursos de todos os namespaces
kubectl get all -A

# Lista todos os serviços do namespace kubernetes-dashboard
kubectl get svc -n kubernetes-dashboard

# Lista todos os serviços do namespace istio-system
kubectl get svc -n istio-system
```

```bash
# Saida do comando: Kubectl get svc -n kubernetes-dashboard
NAME                                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
kubernetes-dashboard-api               ClusterIP   10.98.129.92     <none>        8000/TCP                        4m1s
kubernetes-dashboard-auth              ClusterIP   10.97.165.27     <none>        8000/TCP                        4m1s
kubernetes-dashboard-kong-manager      NodePort    10.108.27.222    <none>        8002:31201/TCP,8445:30841/TCP   4m1s
kubernetes-dashboard-kong-proxy        ClusterIP   10.102.181.148   <none>        443/TCP                         4m1s
kubernetes-dashboard-metrics-scraper   ClusterIP   10.101.169.245   <none>        8000/TCP                        4m1s
kubernetes-dashboard-web               ClusterIP   10.108.130.135   <none>        8000/TCP                        4m1s
```
```bash
# Saída do comando: kubectl get svc -n istio-system
alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP              7m57s
grafana                                   ClusterIP   10.108.69.203    <none>        80/TCP                                  6m47s
istiod                                    ClusterIP   10.108.191.32    <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP   31m
kiali                                     ClusterIP   10.101.38.138    <none>        20001/TCP,9090/TCP                      5m37s
prometheus-grafana                        ClusterIP   10.111.16.100    <none>        80/TCP                                  8m27s
prometheus-kube-prometheus-alertmanager   ClusterIP   10.107.105.190   <none>        9093/TCP,8080/TCP                       8m27s
prometheus-kube-prometheus-operator       ClusterIP   10.100.140.66    <none>        443/TCP                                 8m27s
prometheus-kube-prometheus-prometheus     ClusterIP   10.109.97.83     <none>        9090/TCP,8080/TCP                       8m27s
prometheus-kube-state-metrics             ClusterIP   10.99.146.104    <none>        8080/TCP                                8m27s
prometheus-operated                       ClusterIP   None             <none>        9090/TCP                                7m57s
prometheus-prometheus-node-exporter       ClusterIP   10.109.73.126    <none>        9100/TCP                                8m27s
```
### Expondo o serviços via Nodeport

Os serviços estão acessíveis via ClusterIP que é o tipo de serviço padrão do Kubernetes e só estão disponíveis para acesso internamente no cluster. Para expor os serviços precisaremos abrir uma porta específica em cada nó para cada serviço (NodePort). Criaremos um arquivo chamado `services-nodeport.yaml` com o conteúdo a seguir:

```bash
nano services-nodeport.yaml
```

```bash
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
    release: istio
  name: prometheus-operated-np
  namespace: istio-system
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
    release: istio
  name: grafana-np
  namespace: istio-system
spec:
  type: NodePort
  ports:
    - nodePort: 32366
      port: 80
      protocol: TCP
      targetPort: 3000
  selector:
    app.kubernetes.io/name: grafana
---
# Kiali
apiVersion: v1
kind: Service
metadata:
  labels:
    app: kiali
    release: istio
  name: kiali-np
  namespace: istio-system
spec:
  type: NodePort
  ports:
    - nodePort: 32367
      port: 20001
      protocol: TCP
      targetPort: 20001
  selector:
    app.kubernetes.io/name: kiali
```

Aplique as configurações do arquivo com:

```bash
kubectl apply -f services-nodeport.yaml

# Retorno
service/kubernetes-dashboard-kong-proxy created
service/prometheus-operated-np created
service/grafana-np created
service/kiali-np created
```

Verificando o NodePort dos serviços:

```bash
kubectl get svc -n kubernetes-dashboard

# Retorno
NAME                                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
kubernetes-dashboard-api               ClusterIP   10.98.129.92     <none>        8000/TCP                        12m
kubernetes-dashboard-auth              ClusterIP   10.97.165.27     <none>        8000/TCP                        12m
kubernetes-dashboard-kong-manager      NodePort    10.108.27.222    <none>        8002:31201/TCP,8445:30841/TCP   12m
kubernetes-dashboard-kong-proxy        NodePort    10.107.8.103     <none>        443:32364/TCP                   50s
kubernetes-dashboard-metrics-scraper   ClusterIP   10.101.169.245   <none>        8000/TCP                        12m
kubernetes-dashboard-web               ClusterIP   10.108.130.135   <none>        8000/TCP                        12m
```

```bash
kubectl get svc -n istio-system

# Retorno
NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                 AGE
alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP              25m
grafana                                   ClusterIP   10.108.69.203    <none>        80/TCP                                  24m
grafana-np                                NodePort    10.102.208.91    <none>        80:32366/TCP                            89s
istiod                                    ClusterIP   10.108.191.32    <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP   49m
kiali                                     ClusterIP   10.101.38.138    <none>        20001/TCP,9090/TCP                      23m
kiali-np                                  NodePort    10.108.181.91    <none>        20001:32367/TCP                         89s
prometheus-grafana                        ClusterIP   10.111.16.100    <none>        80/TCP                                  26m
prometheus-kube-prometheus-alertmanager   ClusterIP   10.107.105.190   <none>        9093/TCP,8080/TCP                       26m
prometheus-kube-prometheus-operator       ClusterIP   10.100.140.66    <none>        443/TCP                                 26m
prometheus-kube-prometheus-prometheus     ClusterIP   10.109.97.83     <none>        9090/TCP,8080/TCP                       26m
prometheus-kube-state-metrics             ClusterIP   10.99.146.104    <none>        8080/TCP                                26m
prometheus-operated                       ClusterIP   None             <none>        9090/TCP                                25m
prometheus-operated-np                    NodePort    10.101.240.140   <none>        9090:32365/TCP                          89s
prometheus-prometheus-node-exporter       ClusterIP   10.109.73.126    <none>        9100/TCP                                26m
```

Para acessar cada serviço no navegador utilize os endereços a seguir:

- Kubernetes Dashboard: https://192.168.50.10:32364
- Prometheus: http://192.168.50.10:32365
- Grafana: http://192.168.50.10:32366
- Kiali: http://192.168.50.10:32367

## Acessando os serviços: Kubernetes Dashboard, Prometheus, Grafana e Kiali

Nesta sessão é apresentado os detalhes do acesso a cada uma das ferramentas:

### Kubernetes Dashboard

Ao acessar o Dashboard será apresentada a mensagem de alerta: Potencial risco de segurança à frente. Selecione _Avançado_ e _Aceitar o risco e continuar_. Para acessar o Kubernetes Dasboard precisamos de um usuário e um token de acesso.

![Kubernetes Dashboard](/img/login-dashboard.png)

Faremos isso com o criando um arquivo de implantação user-dashboard.yaml que:

- Cria um usuário para acesso ao Dashboard
- Atribui a função de administrador ao usuário
- Cria um token para autenticação do usuário no Kubernetes Dashboard

O conteúdo do `user-dashboard.yaml`:

```bash
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

# Retorno do comando
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
secret/admin-user-token created
```

Para verificar se o usuário foi criado:

```bash
kubectl -n kubernetes-dashboard get serviceaccount

# Retorno do comando
NAME                                   SECRETS   AGE
admin-user                             0         81s
default                                0         27h
kubernetes-dashboard-api               0         4h8m
kubernetes-dashboard-kong              0         4h8m
kubernetes-dashboard-metrics-scraper   0         4h8m
kubernetes-dashboard-web               0         4h8m
```

Para verificar o token do `admin-user`:

```bash
kubectl -n kubernetes-dashboard describe secret admin-user

# Retorno do comando
Name:         admin-user-token
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 66745383-e8da-4e90-aeae-4eeabaf02aeb

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1107 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IjBBX3kxUTdFX1FmcmlXNTZNOVQ0ODhHeW5raUhHRFVXM3VTal91bElnQzQifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI2Njc0NTM4My1lOGRhLTRlOTAtYWVhZS00ZWVhYmFmMDJhZWIiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.vT9b0ryZPXnl4s1gZCggV7AsDBo4ONirc93JeDirw22YJHjWfPCTKMDa_8Gc_opsN9U_520XCz2x2Xd21ATi6Ux5_px9It9I1qJlBxTu4uGekIK2w6E999m4DiopwigbkGs-y-HBiPWI_XOx45tD22rAm2sJ864ubGgdusiLJZLXHBOtPGdfkrfo3ggYP8lmMVaWPvwVW6dQzQBXcgOhKFBXBLGKnbeCCqWGsege1w0m-sBhD1V12f6xriBJr7Xmj_oCEWeGYivp1RHUSenlEGxb64-HusWcQw5uU815Pu_anOF5dNP-SM0ERo0JIMt7rEjGJQXf8Fu33CzY5xddGw
```

Copie o token e cole na janela de login para ter acesso ao Kubernetes Dashboard.

![kubernetes Dashboard](/img/dashboard.png)

### Prometheus

O Prometheus apresenta sua interface diretamente

![Prometheus](/img/prometheus.png)

### Grafana

Para efetuar login no Grafana utilize o usuário: admin. Para ter acesso a senha utilize o comando:

```bash
kubectl get secret --namespace istio-system grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Copie e cole a senha no campo de Password

![Login Grafana](/img/grafana2.png)

### Kiali

Para acessar o dashboard do kiali será necessário gerar um token de acesso:

```bash
kubectl -n istio-system create token kiali
```

![Acesso ao Kiali](/img/kiali1.png)

Copie e cole a sequência de caracteres no campo Token para ter acesso a interface.

![Acesso ao Kiali](/img/kiali2.png)

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

Esse repoistório é baseia-se nos materiais de: 
- [Alvinditya Saputra](https://github.com/piinalpin/home-lab-provisioning)
- [Javier Ruiz Jiménez](https://github.com/itwonderlab/ansible-vbox-vagrant-kubernetes/tree/master).
- [Vladmir Romanov](https://www.kerno.io/learn/kubernetes-dashboard-deploy-visualize-cluster)