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

Verificando o endereço IP do Master Node:

```bash
kubectl cluster-info
Kubernetes control plane is running at https://192.168.50.10:6443
CoreDNS is running at https://192.168.50.10:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

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
kubectl get all -A
```

## Pacote de Serviços Complementares

Nesta sessão é apresentado a instalação e configuração de ferramentas complementares ao cluster:

- Kugernetes Dashboard
- Prometheus
- Jaeger
- Grafana
- Kiali

### Kubernetes Dashboard

O Kubernetes Dashboard é uma interface web que facilita a gestão de clusters Kubernetes, oferecendo uma visão geral do ambiente, gerenciamento de recursos, solução de problemas e verificação de integridade dos aplicativos. Ao contrário da linha de comando, ele proporciona uma interface visual intuitiva, tornando o monitoramento e a operação do cluster mais acessíveis, especialmente para usuários sem familiaridade com ferramentas de CLI. Isso permite que equipes, como a de suporte de primeira camada, acompanhem a infraestrutura e forneçam respostas rápidas, simplificando a participação no suporte operacional.

Para instalação, no master node execute:

```bash
# Adicionar o repositório do Kubernetes Dashboard
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/

# Atualizar os repositórios do Helm
helm repo update

# Instalar o Dashboard usando Helm
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
```

Teremos o retorno:

```bash
Release "kubernetes-dashboard" does not exist. Installing it now.
NAME: kubernetes-dashboard
LAST DEPLOYED: Tue Sep 10 20:57:30 2024
NAMESPACE: kubernetes-dashboard
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
*************************************************************************************************
*** PLEASE BE PATIENT: Kubernetes Dashboard may need a few minutes to get up and become ready ***
*************************************************************************************************

Congratulations! You have just installed Kubernetes Dashboard in your cluster.

To access Dashboard run:
  kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443

NOTE: In case port-forward command does not work, make sure that kong service name is correct.
      Check the services in Kubernetes Dashboard namespace using:
        kubectl -n kubernetes-dashboard get svc

Dashboard will be available at:
  https://localhost:8443
```

Verificando o serviço criado:

```bash
kubectl -n kubernetes-dashboard get svc

NAME                                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
kubernetes-dashboard-api               ClusterIP   10.96.185.47     <none>        8000/TCP                        2m53s
kubernetes-dashboard-auth              ClusterIP   10.102.43.115    <none>        8000/TCP                        2m53s
kubernetes-dashboard-kong-manager      NodePort    10.98.144.250    <none>        8002:32673/TCP,8445:32013/TCP   2m53s
kubernetes-dashboard-kong-proxy        ClusterIP   10.109.217.122   <none>        443/TCP                         2m53s
kubernetes-dashboard-metrics-scraper   ClusterIP   10.110.131.38    <none>        8000/TCP                        2m53s
kubernetes-dashboard-web               ClusterIP   10.107.21.213    <none>        8000/TCP                        2m53s
```

Caso seja necessário, para remover os componentes aplicados no chart basta:

```bash
helm uninstall kubernetes-dashboard --namespace kubernetes-dashboard
```

O Kubernetes Dashboard está implantado no cluster, porém, por padrão, o tipo de serviço web do dashboard é `ClusterIP`, o que significa que ele so é acessível internamente no cluster. Para acessar o Kubernetes Dashboard da nossa estação de trabalho (fora do cluster), é necessário expor o serviço através de uma prota específica. Essa porta (NodePort) é um número entre 30000 e 32767. Dessa forma o serviço pode ser acessado externamente. Para isso iremos criar um arquivo chamado `kubernetes-dashboard.yaml` que irá atualizar o tipo de serviço do `kubernetes-dashboard-kong-proxy` de `ClusterIP` para `NodePort`. Como também será necessário criar um usuário para acessar o Dashboard iremos inserir essas instruções no mesmo arquivo. O conteúdo do arquivo deverá ser:

```bash
---
kong:
  proxy:
    type: NodePort
    http:
      enabled: true
      nodePort: 30287
```

Agora, atualizaremos a implantação Helm com o arquivo yaml que foi criado:

```bash
helm upgrade kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard -f values.yaml -n kubernetes-dashboard
```

Verificando novamente a situação dos serviços o componente `kubernetes-dashboard-kong-proxy` deverá se apresentar como ClusterIP agora:

```bash
kubectl -n kubernetes-dashboard get svc

NAME                                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
kubernetes-dashboard-api               ClusterIP   10.97.21.239     <none>        8000/TCP                        24m
kubernetes-dashboard-auth              ClusterIP   10.99.151.144    <none>        8000/TCP                        24m
kubernetes-dashboard-kong-manager      NodePort    10.108.141.157   <none>        8002:30449/TCP,8445:30242/TCP   24m
kubernetes-dashboard-kong-proxy        NodePort    10.110.191.177   <none>        443:30286/TCP                   24m
kubernetes-dashboard-metrics-scraper   ClusterIP   10.99.14.162     <none>        8000/TCP                        24m
kubernetes-dashboard-web               ClusterIP   10.103.190.90    <none>        8000/TCP                        24m
```

Agora é possível acessar o Kubernetes Dashboard atrvés do IP do servidor e da porta no nosso caso: http://192.168.50.10:30287/#/login.

![Kubernetes Dashboard](/img/login-dashboard.png)

Para acessar o Kubernetes Dasboard precisamos de um usuário e um token de acesso. Faremos isso com o auxílio do arquivo de implantação user.yaml que:

- Cria um usuário para acesso ao Dashboard
- Atribui a função de administrador ao usuário
- Cria um token para autenticação do usuário no Kubernetes Dashboard

O conteúdo do `user.yaml` deverá ser:

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
kubectl apply -f user.yaml

serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
secret/admin-user-token created
```

Para verificar se o usuário foi criado:

```bash
kubectl -n kubernetes-dashboard get serviceaccount

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
kubectl -n kubernetes-dashboard describe secret admin-user-token
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


### Prometheus, Jaeger, Grafana, and Kiali com NodePort

Para acessar os dashboards através de um navegador web a partir da máquina cliente, é necessário utilizar um proxy, pois os serviços estão escutando em um IP do Cluster.

Durante o desenvolvimento, é possível usar um NodePort para expor cada serviço de maneira insegura.

Crie, no master, o arquivo istio-services-playbook.yaml com o seguinte conteúdo:

```bash
#Grafana
apiVersion: v1
kind: Service
metadata:
  labels:
    app: grafana
    release: istio
  name: grafana-np
  namespace: istio-system
spec:
  ports:
  - name: http
    nodePort: 32493
    port: 3000
    protocol: TCP
    targetPort: 3000
  selector:
    app: grafana
  sessionAffinity: None
  type: NodePort
---
#prometheus
apiVersion: v1
kind: Service
metadata:
  labels:
    app: prometheus
    release: istio
  name: prometheus-np
  namespace: istio-system
spec:
  ports:
  - name: http
    nodePort: 32494
    port: 9090
    protocol: TCP
    targetPort: 9090
  selector:
    app: prometheus
  sessionAffinity: None
  type: NodePort
---
#jaeger
apiVersion: v1
kind: Service
metadata:
  labels:
    app: jaeger
    release: istio
  name: tracing-np
  namespace: istio-system
spec:
  ports:
  - name: http-tracing
    nodePort: 32495
    port: 80
    protocol: TCP
    targetPort: 16686
  selector:
    app: jaeger
  sessionAffinity: None
  type: NodePort
---
#kiali
apiVersion: v1
kind: Service
metadata:
  labels:
    app: kiali
    release: istio
  name: kiali-np
  namespace: istio-system
spec:
  ports:
  - name: http-kiali
    nodePort: 32496
    port: 20001
    protocol: TCP
    targetPort: 20001
  selector:
    app: kiali
  sessionAffinity: None
  type: NodePort
```

Faça o deploy no cluster:

```bash
kubectl apply -f istio-services-playbook.yaml 
service/grafana-np created
service/prometheus-np created
service/tracing-np created
service/kiali-np created
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

Esse repoistório é baseia-se nos materiais de: 
- [Alvinditya Saputra](https://github.com/piinalpin/home-lab-provisioning)
- [Javier Ruiz Jiménez](https://github.com/itwonderlab/ansible-vbox-vagrant-kubernetes/tree/master).
- [Vladmir Romanov](https://www.kerno.io/learn/kubernetes-dashboard-deploy-visualize-cluster)