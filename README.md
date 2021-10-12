# Kubernetes_CentOs

Segue abaixo o passo a passo para instalação da versão 1.18 do Kubernetes no CentOS 8.

Para o caso abaixo, é realizado instalação do Kubernetes em quatro servidores. Um deles o master ( controll plane ) e outros três os workers.
No Kubernetes o nó master é responsável por executar serviços do próprio Kubernetes e garantir que o estado do cluster seja estável. É o master quem verifica o estado atual e realiza o agendamento dos containers para execução nos workers.
Já os workers são responsáveis por executar os pods. É no worker onde os famosos pods ficam.


Configurações iniciais

Estas configurações deverão ser realizadas em todos os servidores que farão parte do cluster, seja ele master ou worker e visa preparar tudo para a instalação do Kubernetes.

Para iniciar, tome nota do nome de cada servidor do cluster.

Caso não saiba os nomes das máquinas você pode utilizar o seguinte comando para descobrir:

$ hostname

Em cada um dos nós, edite o arquivo /etc/hosts e configure os ips de cada uns dos hosts.

$ sudo vi /etc/hosts

Desabilite o SELinux do CentOS que vem habilitador por padrão.

$ sudo setenforce 0

$ sudo sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

Configure o firewall para permitir livre comunicação do Kubernetes.

$ sudo firewall-cmd --permanent --add-port=6443/tcp

$ sudo firewall-cmd --permanent --add-port=2379-2380/tcp

$ sudo firewall-cmd --permanent --add-port=10250/tcp

$ sudo firewall-cmd --permanent --add-port=10251/tcp

$ sudo firewall-cmd --permanent --add-port=10252/tcp

$ sudo firewall-cmd --permanent --add-port=10255/tcp

$ sudo firewall-cmd --reload

$ sudo modprobe br_netfilter

$ sudo su 

$ echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables

$ exit

O Kubernetes não trabalha bem com o Swap. Por isso, desabilite-o em todos os nós:

$ sudo swapoff -a

Para que o Swap não seja habilitado automaticamente no boot, edite o arquico /etc/fstab executando o seguinte comando:

$ sudo sed -i 's/.*swap.*/#&/' /etc/fstab

Instalando o Docker

Após realizadas as configurações inicias, é necessario instalar o Docker nos servidores.

Para isso, execute os seguintes comandos (em todos os servidores):

$ sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

$ sudo dnf -y install https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm

$ sudo dnf install -y docker-ce

$ sudo systemctl enable docker

$ sudo systemctl start docker

$ sudo usermod -aG docker $USER

Após essa etapa é necessário sair e logar novamente com o usuário para que as configurações ao usuário sejam aplicadas.
Para testar o pleno funcionamento do Docker execute o seguinte comando:

$ docker run hello-world

Se tudo ocorrer bem, a imagem do hello-world deverá ser baixada e executada no host. 

Instalação do Kuberenetes

Com o Docker instalado e todas configurações prontas, é hora de finalmente instalar o Kubernetes. Esses passos devem também serem realizados em todos os nós.
Para isso, é necessário primeiro configurar o repositório do Kubernetes. Para isso, execute:

$ sudo su

$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
  
$ exit
  
Em seguida, realize a instalação:
  
$ sudo dnf install kubeadm-1.18.6 -y
  
$ sudo systemctl enable kubelet
  
$ sudo systemctl start kubelet
  
Estamos quase lá, precisamos agora configurar o control-pane do Kubernetes que deve ser feito apenas no nó master.
  
Configurando o master
  
Esses passos devem ser executados apenas o nó master, para criar o painel de controle do cluster.
Inicie o cluster:
  
$ sudo kubeadm init
  
Tome nota do comando destacado de vermelho. Ele será utilizado logo a seguir!
  
O próximo passo deve ser executado no seu usuário comum. Perceba que em todo procedimento é utilizado o sudo para elevar as permissões. Se você utilizou o atalho de logar diretamente com o root, agora é a hora de voltar ao seu usuário. Feito isso, execute os seguintes comandos:
  
$ mkdir -p $HOME/.kube
  
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
Confirme que tudo está funcionando com o seguinte comando:
  
$ kubectl get nodes
  
Neste momento, é normal o host retornar como “NotReady”. Isso ocorre por que precisamos instalar e configurar a rede do Kubernetes. Existem vários tipos de redes suportadas pelo Kubernetes. Neste tutorial, utilizaremos a rede Weave.
  
Aqui vale um pequeno comentário: Em vez de utilizar o comando padrão do weave (que muda a cada nova versão), preferi baixar o yaml da versão testada e disponibilizar no meu repositório.
  
Isso é particularmente importante, pois é uma forma de garantir que o procedimento funcione em qualquer momento do tempo. Outro detalhe: foi necessário fazer um ajuste, conforme reportado nesta issue.
  
Então, para iniciar, execute o seguinte comando:
  
$ git clone https://github.com/dudures/weaves-kubernetes.git
  
$ kubectl apply -f kubernetes-weaves/weave-2.6.5.yaml
  
Se preferir utilizar a última versão do Weave, execute o seguinte comando, conforme documentação:
  
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
  
Espere alguns segundos (ou 1 minuto) e digite nomamente o seguinte comando:
  
$ kubectl get nodes
  
Decorrido um tempinho, o nó master deve passar para o estado de Ready.
  
Estando tudo certo no nó master, vamos agora adicionar os demais nós workers no cluster:
Nos nós workers
Lembra do comando de join que destacamos em vermelho quando iniciamos o cluster?
  
É necessário executá-lo em cada worker para que o worker possa ingressar no cluster.
  
No caso, o seguinte comando foi gerado:
  
sudo kubeadm join 10.3.224.209:6443 --token lhxbs4.i8qxsph897aqjyt2 --discovery-token-ca-cert-hash sha256:55d28ceb976611d80a1f4630e30e6981ee00a385a65d5830ec3866c5bc14a89f
  
Para ter certeza que o nó entrou no cluster volte ao nó master e execute o seguinte comando:
  
$ kubectl get nodes
  
Este comando mostrará os nós disponíveis no cluster. Ao final, o resultado esperado é o seguinte:

Caso não tenha tomado nota do comando de join, ou por acaso ele tenha expirado, você pode gerar um novo comando de join, executando o seguinte comando no master:
  
kubeadm token create --print-join-command
  
