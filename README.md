


graph TD
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
