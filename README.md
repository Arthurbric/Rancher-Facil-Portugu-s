

---

![rgs-rancher-banner](images/rgs-rancher-banner.png)

# Instalação Fácil de RKE2, Rancher Manager, Longhorn e NeuVector

### Índice

- [Sobre mim](#sobre-mim)
- [Introdução](#introducao)
- [Assista ao vídeo](https://www.youtube.com/watch?v=47ZcCMgKNWw)
- [Infraestrutura](#infraestrutura)
- [Rancher Kubernetes (RKE2)](#rancher-kubernetes-rke2)
- [Rancher Multi Cluster Manager](#rancher-multi-cluster-manager)
- [Rancher Longhorn](#rancher-longhorn)
- [Rancher NeuVector](#rancher-neuvector)
- [Conclusão](#conclusão)

## Sobre mim

Um pouco da minha história...

- Arquiteto de Soluções na SUSE México
- Trabalho na área de pré-vendas há mais de 8 anos
- Conhecimento em ambientes on-premise, nuvem e agora cloud-native
- Gamer de nascença, adoro jogar muitos jogos dos anos 90!

## Introdução {#introducao}


### Bem-vindo ao Guia de Instalação Fácil do Rancher.

Neste guia de implementação, instalaremos todo o stack da SUSE Rancher, que inclui os seguintes produtos:

- RKE2 (Distribuição Kubernetes) - [Clique aqui para saber mais](https://ranchergovernment.com/products/rke2)
- Rancher Manager (Gerenciamento de Clusters) - [Clique aqui para saber mais](https://ranchergovernment.com/products/mcm)
- Longhorn (Armazenamento) - [Clique aqui para saber mais](https://www.ranchergovernment.com/products/longhorn)
- NeuVector (Segurança) - [Clique aqui para saber mais](https://ranchergovernment.com/neuvector)

### Pré-requisitos

- Três (3) servidores Linux com acesso à internet
- Ferramentas para gerenciar os servidores (Terminal, VSCode, Termius, etc.)

## Assista ao vídeo

Se preferir seguir este guia com um incrível vídeo... por favor, clique abaixo. (https://www.youtube.com/watch?v=47ZcCMgKNWw)!

[![rancher-effortless-youtube-video](images/rancher-effortless-thumbnail.png.png)]()

## Infraestrutura

Para essa implementação, precisaremos de três servidores Linux para colocar tudo em funcionamento. Estaremos utilizando três servidores OpenSUSE Leap 15.5 virtualizados, provisionados via VirtualBox. Qualquer distribuição Linux deve funcionar, desde que haja conectividade de rede. Aqui está uma lista dos nossos [Sistemas Operacionais suportados](https://docs.rke2.io/install/requirements#operating-systems). Para configurar esses servidores para Rancher, eles precisam estar conectados à internet e acessíveis via `ssh`.

Aqui está uma visão geral da arquitetura que usaremos neste guia de implementação:

![rancher-harvester-vm-overview](images/rancher-harvester-vm-overview.png)

Vamos executar os seguintes comandos em cada um dos nós para garantir que eles tenham os pacotes e configurações necessárias.

```bash
# servidores: rke2-cp-01, rke2-wk-01 e rke2-wk-02
# Instalar pacotes
zypper --non-interactive install -n open-iscsi && systemctl enable iscsid && systemctl start iscsid

# Desativar o Firewall
systemctl stop firewalld && systemctl disable firewalld
```


## Rancher Kubernetes (RKE2)

Para configurar e instalar o RKE2, é necessário ter nós de **Controle** e nós **Worker**. Começaremos configurando o nó de Controle e depois os nós Worker. Existem várias formas de realizar essa configuração, e esta guia é projetada para uma instalação mínima e fácil. Consulte a [documentação do RKE2](https://docs.rke2.io) para obter mais informações.

### RKE2 Nodo de Controle

Começaremos configurando o nó de Controle RKE2, adicionando um arquivo de configuração. Nesta instalação simples, utilizaremos a configuração por token para o RKE2. A sessão será feita via SSH com root para o servidor `rke2-cp-01`.

Caso deseje explorar outras formas de configuração do nó de Controle, veja os [documentos do servidor RKE2](https://docs.rke2.io/reference/server_config).

**Passos para o controle:**

```bash
# Criar o diretório RKE2
mkdir -p /etc/rancher/rke2/

# Criar o arquivo de configuração RKE2
cat << EOF >> /etc/rancher/rke2/config.yaml
token: rke2SecurePassword
EOF
```

Agora que o arquivo de configuração está completo, vamos instalar e iniciar o nó de Controle RKE2:

```bash
# Baixar e instalar o RKE2 no modo Controle
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.30 INSTALL_RKE2_TYPE=server sh -

# Iniciar o serviço de Controle RKE2
systemctl enable rke2-server.service && systemctl start rke2-server.service
```

Para verificar o funcionamento do nó de Controle:

```bash
# Verificar o status do serviço
systemctl status rke2-server
```
Vamos verificar se o nó de controle está em execução usando systemctl status rke2-server. Deve aparecer assim:

![rancher-rke2-cp-01-systemctl](images/rancher-rke2-cp-01-systemctl.png)
---
Agora, vamos verificar usando `kubectl`:

```bash
# Criar link simbólico para kubectl e containerd
sudo ln -s /var/lib/rancher/rke2/data/v1*/bin/kubectl /usr/bin/kubectl
sudo ln -s /var/run/k3s/containerd/containerd.sock /var/run/containerd/containerd.sock

# Atualizar BASHRC
cat << EOF >> ~/.bashrc
export PATH=$PATH:/var/lib/rancher/rke2/bin:/usr/local/bin/
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export CRI_CONFIG_FILE=/var/lib/rancher/rke2/agent/etc/crictl.yaml
alias k=kubectl
EOF

# Aplicar BASHRC
source ~/.bashrc

# Verificar que o RKE2 está funcionando
kubectl get nodes
```
Deveria aparecer assim:

![rancher-rke2-cp-01-kubectl](images/rancher-rke2-cp-01-kubectl.png)
---

## RKE2 Nodos Worker

Agora, vamos configurar os nós de Worker do RKE2 adicionando o arquivo de configuração. Como estamos realizando uma instalação simples, utilizaremos a opção de configuração por token para o RKE2 e configuraremos os Workers. Estou em uma sessão SSH como root para acessar os servidores rke2-wk-01 e rke2-wk-02.

Se você deseja explorar outras formas de configurar o nó Worker do RKE2, consulte a [documentação do agente RKE2](https://docs.rke2.io/reference/linux_agent_config).

**Nota:** Esses passos devem ser realizados em cada nó de Worker, e não se esqueça de ajustar o endereço IP do nó de Controle de acordo com a configuração que você está utilizando.


```bash
# Criar o diretório RKE2
mkdir -p /etc/rancher/rke2/

# Criar o arquivo de configuração RKE2 para os nós Worker
cat << EOF >> /etc/rancher/rke2/config.yaml
server: https://10.0.0.15:9345
token: rke2SecurePassword
EOF
```

Instale e inicie o serviço nos nós Worker:

```bash
# Baixar e instalar o RKE2 no modo Worker
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.30 INSTALL_RKE2_TYPE=agent sh -

# Iniciar o serviço dos nós Worker
systemctl enable rke2-agent.service && systemctl start rke2-agent.service
```

Retorne ao servidor `rke2-cp-01` para verificar se os nós Worker foram unidos ao cluster:

```bash
# Verificar que RKE2 está funcionando
kubectl get nodes
```

Deveria aparecer assim:

![rancher-rke2-cp-01-kubectl-all](images/rancher-rke2-cp-01-kubectl-all.png)

Parabéns! Agora você tem seu cluster RKE2 em funcionamento! Se você já está familiarizado com Kubernetes ou RKE2, sinta-se à vontade para explorar o cluster usando o "kubectl". Agora, vamos instalar o [Rancher Multi Cluster Manager](https://www.ranchergovernment.com/products/mcm), [Rancher Longhorn](https://www.ranchergovernment.com/products/longhorn) e [Rancher NeuVector](https://ranchergovernment.com/neuvector).

---

## Rancher Multi Cluster Manager

Quando a maioria das pessoas começa sua jornada com Kubernetes e Rancher Kubernetes, há uma certa confusão sobre as camadas do Kubernetes. O RKE2 é a nossa distribuição de Kubernetes, enquanto o Rancher Multi Cluster Manager é o painel de controle que usamos para gerenciar qualquer tipo de cluster Kubernetes (incluindo aqueles listados pela CNCF). Para executar nosso Rancher Manager, primeiro precisávamos de um cluster Kubernetes, e é por isso que começamos com a instalação do RKE2.

Vamos começar a instalação do Rancher Manager! Para obter os componentes necessários para configurá-lo e instalá-lo, precisamos usar o [Helm CLI](https://helm.sh) como gerenciador de pacotes (charts) para Kubernetes, instalar o chart do [Cert Manager](https://cert-manager.io), e finalmente o chart do Rancher Manager. Vamos usar `ssh` com `root` para acessar o servidor `rke2-cp-01` e executar os seguintes comandos:

```bash
# Servidor(s): rke2-cp-01
# Baixar e instalar o Helm
mkdir -p /opt/rancher/helm
cd /opt/rancher/helm

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 755 get_helm.sh && ./get_helm.sh
mv /usr/local/bin/helm /usr/bin/helm
```

Agora, vamos adicionar os repositórios do Helm para Cert Manager e Rancher Manager:

```bash
# Servidor(s): rke2-cp-01
# Adicionar e atualizar os repositórios do Helm
helm repo add jetstack https://charts.jetstack.io
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
```

Deveria ficar assim:

![rancher-helm-repo-status](images/rancher-helm-repo-status.png)

Agora, vamos instalar o Cert Manager com os seguintes comandos:

```bash
# Servidor(s): rke2-cp-01
# Criar o namespace do Cert Manager e depois instalar o Cert Manager
kubectl create namespace cert-manager

helm upgrade -i cert-manager jetstack/cert-manager --namespace cert-manager --set installCRDs=true

# Aguarde a implantação e inicialização.
sleep 60

# Verificar o status do Cert Manager
kubectl get pods --namespace cert-manager
```

Deveria ficar assim:

![rancher-cert-manager-status](images/rancher-cert-manager-status.png)

Agora, vamos instalar o Rancher Manager com os seguintes comandos (Observe o hostname que configuramos nos comandos; se preferir, você pode alterar esse campo, assim como a senha):

```bash
# Servidor(s): rke2-cp-01
# Criar o namespace do Rancher e depois instalar o Rancher
kubectl create namespace cattle-system

helm upgrade -i rancher rancher-stable/rancher --namespace cattle-system --set bootstrapPassword=rancherSecurePassword --set hostname=rancher.10.0.0.15.sslip.io

# Aguarde a implantação e inicialização.
sleep 45

# Verificar o status do Rancher Manager
kubectl get pods --namespace cattle-system
```

Deveria ficar assim:

![rancher-rancher-manager-status](images/rancher-rancher-manager-status.png)
---

## Explorando o Rancher Manager

Depois que todos os pods estiverem no estado `Running` (Em execução) no namespace `cattle-system`, você pode acessar o Rancher Manager! Como estamos usando `sslip.io` como nosso nome de host/DNS, não é necessário configurar mais nada para acessar o Rancher Manager. Vamos verificar o hostname e dar uma olhada no Rancher Manager!

Para esta implementação, usamos `https://rancher.10.0.0.15.sslip.io` para acessar o Rancher Manager. No seu caso, verifique qual foi o nome de host configurado na etapa anterior.

Deve parecer assim:

![rancher-rancher-manager-bootstrap](images/rancher-rancher-manager-bootstrap.png)

![rancher-rancher-manager-terms](images/rancher-rancher-manager-terms.png)

Agora você deve visualizar o Rancher Manager solicitando a senha que configuramos durante a instalação. Para esta implementação, usei `rancherSecurePassword`. Você também precisará verificar a URL do Rancher Manager e aceitar os Termos e Condições. Depois de concluir... Deve parecer assim:

![rancher-rancher-manager-home](images/rancher-rancher-manager-home.png)

Agora você tem o Rancher Manager implementado com sucesso em nosso cluster RKE2 Kubernetes!!! Lembre-se de que existem várias maneiras de configurá-lo, e esta foi apenas uma instalação mínima e simples. Sinta-se à vontade para explorar tudo o que pode fazer dentro do Rancher Manager. Neste caso, passaremos para a próxima etapa de instalação do Rancher Longhorn.

## Rancher Longhorn

Vamos adicionar o repositório do Helm para Longhorn:

```bash
# servidor(s): rke2-cp-01
# Adicionar e atualizar o repositório do Helm
helm repo add longhorn https://charts.longhorn.io
helm repo update
```

Deve parecer assim:

![rancher-helm-repo-status-longhorn](images/rancher-helm-repo-status-longhorn.png)

Agora, vamos instalar o Longhorn com os seguintes comandos (Lembre-se de revisar os comandos caso deseje modificar opções, como o hostname):

```bash
# servidor(s): rke2-cp-01
# Criar o namespace do Longhorn e instalar o Longhorn
kubectl create namespace longhorn-system

helm upgrade -i longhorn longhorn/longhorn --namespace longhorn-system --set ingress.enabled=true --set ingress.host=longhorn.10.0.0.15.sslip.io

# Aguarde a implementação e o lançamento.
sleep 60

# Verificar o status do Longhorn
kubectl get pods --namespace longhorn-system
```

Debería verse así:

![rancher-longhorn-status](images/rancher-longhorn-status.png)
---


### Explorando o Rancher Longhorn

Assim que todos os pods estiverem no estado `Running` (Em execução) no namespace `longhorn-system`, você poderá acessar o Rancher Longhorn! Assim como no Rancher Manager, utilizamos `sslip.io`, portanto, não é necessário fazer configurações adicionais para acessar o Longhorn. Vamos ao nome de domínio.

Para esta implementação, utilizamos `https://longhorn.10.0.0.15.sslip.io` para acessar o Rancher Longhorn.

Deve parecer assim:

![rancher-longhorn-home](images/rancher-longhorn-home.png)

Agora você tem o Rancher Longhorn implementado com sucesso em nosso cluster RKE2 com o Rancher Manager! Sinta-se à vontade para explorar o painel do Longhorn e ver como é fácil gerenciar volumes, fazer backups em um bucket S3 ou configurar recuperação de desastres entre clusters. Por enquanto, vamos passar para a instalação do Rancher NeuVector.

## Rancher NeuVector

Vamos adicionar o repositório Helm para o NeuVector!

```bash
# servidor(s): rke2-cp-01
# Adicionar e atualizar o repositório do Helm
helm repo add neuvector https://neuvector.github.io/neuvector-helm
helm repo update
```

Deve parecer assim:

![rancher-helm-repo-status-neuvector](images/rancher-helm-repo-status-neuvector.png)

Agora, vamos instalar o NeuVector com os seguintes comandos (Lembre-se de revisar os comandos caso deseje modificar opções, como o hostname):

```bash
# servidor(s): rke2-cp-01
# Criar o namespace do NeuVector e instalar o NeuVector
kubectl create namespace cattle-neuvector-system

helm upgrade -i neuvector neuvector/core --namespace cattle-neuvector-system --set k3s.enabled=true --set manager.ingress.enabled=true --set manager.svc.type=ClusterIP --set controller.pvc.enabled=true --set manager.ingress.host=neuvector.10.0.0.15.sslip.io --set global.cattle.url=https://rancher.10.0.0.15.sslip.io --set controller.ranchersso.enabled=true --set rbac=true

# Aguarde a implementação e o lançamento
sleep 60

# Verificar o status do NeuVector
kubectl get pods --namespace cattle-neuvector-system
```

Deve parecer assim:

![rancher-neuvector-status](images/rancher-neuvector-status.png)

## Explorando o Rancher NeuVector

Assim que todos os pods estiverem no estado `Running` (Em execução) no namespace "cattle-neuvector-system", você poderá acessar o NeuVector. Assim como no Rancher Manager e Rancher Longhorn, utilizamos `sslip.io`, então não é necessário fazer mais configurações para acessar o NeuVector. Vamos ao nome de domínio.

Para esta implementação, utilizamos `https://neuvector.10.0.0.15.sslip.io` para acessar o NeuVector.

Deve parecer assim:

![rancher-neuvector-bootstrap](images/rancher-neuvector-bootstrap.png)

Agora você deverá ver o NeuVector solicitando o nome de usuário e senha padrão. O nome de usuário padrão é "admin" e a senha padrão é "admin".

Deve parecer assim:

![rancher-neuvector-home](images/rancher-neuvector-home.png)

Agora você tem o Rancher NeuVector implementado no nosso cluster RKE2 com Rancher Manager e Rancher Longhorn! Sinta-se à vontade para explorar o NeuVector, executar análises de vulnerabilidade, investigar os componentes do cluster ou verificar a atividade da rede do seu Kubernetes. Aqui, normalmente recomendamos aos usuários que tentem criar um novo cluster ou implementar algumas aplicações de teste para ver o verdadeiro poder do Rancher. Por enquanto, vamos passar para a nossa Conclusão...

## Conclusão

Em poucos passos simples e em questão de minutos, conseguimos implementar todo o stack do Rancher, pronto para ser explorado e utilizado. A facilidade com que instalamos esses componentes é o que torna essa jornada tão empolgante e acessível, por isso batizei este guia de "Rancher fácil".

Se você encontrar algum desafio durante a implementação, estarei à disposição para ajudar! Lembre-se, cada passo dado é um avanço na jornada de dominar o Kubernetes com o Rancher. Obrigado por seguir até aqui, e que essa experiência seja apenas o começo de muitas conquistas no mundo da tecnologia.

Continue explorando, aprendendo e alcançando novos horizontes! Até breve! 🌟
