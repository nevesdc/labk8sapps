

Documentação: Laboratório Kubernetes em Hyper-V (Rede 192.168.3.0/24)1. Visão GeralEste documento descreve a arquitetura e a configuração de um cluster Kubernetes (K8s) "bare-metal" (em VMs), construído sobre o hipervisor Hyper-V. O ambiente foi projetado para testes de failover e aplicações "stateful", utilizando Kubeadm, Calico, MetalLB e um servidor NFS dedicado para armazenamento persistente.A rede do laboratório é a 192.168.3.0/24.2. Arquitetura e Planejamento de IPsA arquitetura consiste em um único Virtual Switch no Hyper-V conectado a 5 VMs com IPs estáticos. O MetalLB gerencia um pool de IPs virtuais para serviços externos.Planejamento de IPsVM / ServiçoHostnameIP EstáticoPropósitoControl Planek8s-cp192.168.3.120Gerencia o cluster K8s (API, etcd)Worker 1k8s-w1192.168.3.121Executa aplicações (Pods)Worker 2k8s-w2192.168.3.122Executa aplicações (Pods)Worker 3k8s-w3192.168.3.123Executa aplicações (Pods)NFS Serverk8s-nfs192.168.3.240Armazenamento persistente (Storage)RedeN/A192.168.3.0/24Sub-rede do laboratórioGatewayN/A192.168.3.254Roteador da redeMetalLB PoolN/A192.168.3.200-210IPs virtuais para serviços K8sDiagrama da ArquiteturaSnippet de códigograph TD
    subgraph LAN (Rede 192.168.3.0/24)
        direction LR
        USER[💻 Usuário]
        HOST[
            <b>Hyper-V Host</b>
            <br/>(Servidor Físico)
        ]
        GW(<b>Gateway</b><br/>192.168.3.254)

        USER -- Acesso --> LB_IP
    end

    subgraph HOST
        direction TB
        VNET(🌐 Virtual Switch Externo)

        subgraph K8S_CLUSTER [Cluster Kubernetes]
            direction LR
            CP(<b>k8s-cp</b><br/>192.168.3.120)
            W1(<b>k8s-w1</b><br/>192.168.3.121)
            W2(<b>k8s-w2</b><br/>192.168.3.122)
            W3(<b>k8s-w3</b><br/>192.168.3.123)
        end

        NFS_VM(<b>k8s-nfs</b><br/>192.168.3.240)

        VNET -- Conecta --> CP
        VNET -- Conecta --> W1
        VNET -- Conecta --> W2
        VNET -- Conecta --> W3
        VNET -- Conecta --> NFS_VM
    end

    subgraph Lógica do Cluster
        direction TB
        LB_IP(<b>[ MetalLB ]</b><br/>Pool: 192.168.3.200-210<br/>Load Balancer)

        subgraph K8S_WORKLOADS [Aplicações]
            direction LR
            APP1(<b>App: "Onde estou?"</b><br/>(3 réplicas))
            APP2(<b>App: "Writers/Reader"</b><br/>(4 pods))
        end

        subgraph K8S_STORAGE [Armazenamento]
            direction TB
            PROV(<b>NFS Provisioner</b>)
            SC[<b>StorageClass 'nfs-storage'</b><br/>(ReadWriteMany)]
        end

        LB_IP -- Roteia para --> APP1
        APP2 -- Monta PVC (RWX) --> SC
        SC -- Usa --> PROV
        PROV -- Monta (NFS) --> NFS_VM
    end
3. Passo a Passo: Instalação e ConfiguraçãoEsta seção detalha a criação das VMs e a instalação de todos os componentes necessários.3.1. Pré-requisitos (Hyper-V)Hipervisor: Microsoft Hyper-V instalado e funcional.Rede: Um "Virtual Switch" do tipo External criado no Hyper-V Manager, conectado à sua placa de rede física.ISO: ISO do Ubuntu Server 22.04 LTS.VMs: Crie 5 novas VMs (1x k8s-cp, 3x k8s-wX, 1x k8s-nfs), conecte todas ao Virtual Switch External e instale o Ubuntu 22.04 em todas.3.2. Configuração Base do S.O. (Ubuntu 22.04)Execute em TODAS as 5 VMs (cp, w1, w2, w3, nfs)Definir IP Estático (Netplan):O Ubuntu 22.04 usa netplan. Edite o arquivo YAML de configuração (ex: /etc/netplan/01-netcfg.yaml).Exemplo de template para k8s-cp (192.168.3.120):YAMLnetwork:
  ethernets:
    eth0: # O nome da sua interface pode variar (ex: ens33)
      dhcp4: no
      addresses:
        - 192.168.3.120/24
      routes:
        - to: default
          via: 192.168.3.254
      nameservers:
        addresses: [192.168.3.254, 8.8.8.8] # Gateway e Google DNS
  version: 2
Importante: Adapte o campo addresses para cada nó (.121, .122, .123, .240).Aplique a configuração: sudo netplan applyDefinir Hostname:Bash# Exemplo para k8s-cp
sudo hostnamectl set-hostname k8s-cp
# (Repita em todos os nós com seus respectivos nomes)
Atualizar o Sistema:Bashsudo apt update
sudo apt upgrade -y
Carregar Módulos de Kernel (Apenas nos 4 nós K8s):Bashcat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
Configurar Sysctl (Apenas nos 4 nós K8s):Bashcat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
3.3. Instalação do Servidor NFS (k8s-nfs)Execute APENAS na VM k8s-nfs (192.168.3.240)Instalar Servidor NFS:Bashsudo apt update
sudo apt install -y nfs-kernel-server
Criar Diretório de Compartilhamento:Bashsudo mkdir -p /export/k8s-data
sudo chown nobody:nogroup /export/k8s-data
sudo chmod 777 /export/k8s-data
Configurar Exportação:Edite o arquivo /etc/exports:Bashsudo nano /etc/exports
Adicione esta linha (autorizando a sua rede 192.168.3.0/24):/export/k8s-data    192.168.3.0/24(rw,sync,no_subtree_check,no_root_squash)
Aplicar Configurações:Bashsudo exportfs -a
sudo systemctl restart nfs-kernel-server
3.4. Instalação do Runtime (containerd)Execute APENAS nos 4 nós K8s (cp, w1, w2, w3)Instalar containerd:Bashsudo apt install -y containerd
Configurar cgroup:Bash# Criar arquivo de configuração padrão
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Habilitar o SystemdCgroup (CRÍTICO)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
Reiniciar containerd:Bashsudo systemctl restart containerd
3.5. Instalação do Kubeadm e KubeletExecute APENAS nos 4 nós K8s (cp, w1, w2, w3)Desabilitar Swap (Requisito do Kubelet):Bashsudo swapoff -a
# Desabilita permanentemente
sudo sed -i '/swap/d' /etc/fstab
sudo rm /swap.img 2>/dev/null || sudo rm /swapfile 2>/dev/null
Adicionar Repositório K8s:Bashsudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg

# Adicionar chave GPG
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Adicionar repositório
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
Instalar Ferramentas K8s:Bashsudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
4. Passo a Passo: Inicialização do Cluster4.1. Inicialização do Control PlaneExecute APENAS no k8s-cp (192.168.3.120)Bashsudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --control-plane-endpoint=192.168.3.120
Guarde o comando kubeadm join que aparecerá no final.Para usar o kubectl como usuário normal:Bashmkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
4.2. Instalação do CNI (Calico)Execute APENAS no k8s-cpBashkubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
Aguarde alguns minutos. O nó k8s-cp mudará para o status Ready (kubectl get nodes).4.3. Adição dos Worker NodesExecute APENAS em k8s-w1, k8s-w2, k8s-w3Execute o comando kubeadm join (com sudo) que você salvou do passo 4.1.Bash# Exemplo
sudo kubeadm join 192.168.3.120:6443 --token <seu-token> \
    --discovery-token-ca-cert-hash sha256:<seu-hash>
No k8s-cp, execute kubectl get nodes e observe os 3 workers entrarem no cluster com o status Ready.5. Configuração dos Componentes do Cluster5.1. Load Balancer (MetalLB)Execute APENAS no k8s-cpInstalar MetalLB:Bashkubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.7/config/manifests/metallb-native.yaml
Aguarde os pods em metallb-system ficarem Running.Configurar Pool de IPs:Crie um arquivo metallb-pool.yaml com o pool 192.168.3.200-210:YAMLapiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.3.200-192.168.3.210
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
Aplique o arquivo: kubectl apply -f metallb-pool.yaml5.2. Armazenamento (NFS Provisioner)Execute APENAS no k8s-cpInstalar Cliente NFS (em todos os nós K8s):Execute em k8s-cp, k8s-w1, k8s-w2, k8s-w3:Bashsudo apt update
sudo apt install -y nfs-common
Instalar Provisionador (Helm):Bashhelm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --namespace nfs-provisioner \
    --create-namespace \
    --set nfs.server=192.168.3.240 \
    --set nfs.path=/export/k8s-data \
    --set storageClass.name=nfs-storage \
    --set storageClass.defaultClass=true
Nota: nfs.server aponta para o IP do seu k8s-nfs.5.3. Otimização de FailoverObjetivo: Reduzir o tempo de detecção de falha de nó (padrão de 5+ min).Problema Identificado: Muitas flags de kube-controller-manager (como --pod-eviction-timeout) estão obsoletas na v1.30 e causam CrashLoopBackOff.Solução Final: Apenas a flag --node-monitor-grace-period foi adicionada ao arquivo /etc/kubernetes/manifests/kube-controller-manager.yaml no nó k8s-cp.YAML# Trecho do /etc/kubernetes/manifests/kube-controller-manager.yaml
...
  containers:
  - command:
    - kube-controller-manager
...
    # FLAG ADICIONADA PARA DETECÇÃO RÁPIDA:
    - --node-monitor-grace-period=10s
...
6. Aplicações de Exemplo e Testes6.1. App 1: "Onde estou?" (Stateless)Teste: Um Deployment de 3 réplicas com um Service (LoadBalancer) foi criado. A Downward API injetou o nome do nó (spec.nodeName) em uma variável de ambiente ($K8S_NODE_NAME).Resultado: Acessar o IP do Load Balancer (ex: http://192.168.3.200) e executar um curl em loop demonstrou o balanceamento de carga round-robin entre os pods nos diferentes nós (k8s-w1, k8s-w2, k8s-w3).6.2. App 2: "Writers/Reader" (Stateful, NFS)Teste: Um PVC com modo ReadWriteMany (RWX) foi criado. Um Deployment de 3 pods ("Writers") e 1 pod ("Reader") montaram o mesmo PVC.Resultado: Os "Writers" escreveram em um arquivo de log compartilhado (/data/log.txt). Os logs do "Reader" (kubectl logs -f reader-app) mostraram os dados dos 3 escritores sendo gravados em tempo real, validando o armazenamento RWX.6.3. Correção de Fuso Horário (Timezone)Problema: Os timestamps dos pods (ex: busybox) estavam em UTC, mesmo com os nós em GMT-3.Solução:A imagem do contêiner foi alterada de busybox para ubuntu:latest (que inclui tzdata).O fuso horário do host (/etc/localtime) foi montado no contêiner via hostPath para garantir o offset de fuso correto.YAML# Trecho da solução de Timezone no Deployment
      containers:
      - name: writer
        image: ubuntu:latest # 1. Imagem completa
        volumeMounts:
        - name: nfs-volume
          mountPath: "/data"
        - name: host-time
          mountPath: /etc/localtime # 2. Montagem do fuso do host
          readOnly: true
      volumes:
      - name: nfs-volume
        persistentVolumeClaim:
          claimName: nfs-shared-pvc
      - name: host-time
        hostPath:
          path: /etc/localtime
          type: File
