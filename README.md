# Documentação: Laboratório Kubernetes em Hyper-V (Rede 192.168.3.0/24)

## 1. Visão Geral

Este documento descreve a arquitetura e a configuração de um cluster Kubernetes (K8s) "bare-metal" (em VMs), construído sobre o hipervisor Hyper-V. O ambiente foi projetado para testes de failover e aplicações "stateful", utilizando Kubeadm, Calico, MetalLB e um servidor NFS dedicado para armazenamento persistente.

A rede do laboratório é a **`192.168.3.0/24`**.

## 2. Arquitetura e Planejamento de IPs

A arquitetura consiste em um único Virtual Switch no Hyper-V conectado a 5 VMs com IPs estáticos. O MetalLB gerencia um pool de IPs virtuais para serviços externos.

### Planejamento de IPs
| VM / Serviço | Hostname | IP Estático | Propósito |
| :--- | :--- | :--- | :--- |
| Control Plane | `k8s-cp` | `192.168.3.120` | Gerencia o cluster K8s (API, etcd) |
| Worker 1 | `k8s-w1` | `192.168.3.121` | Executa aplicações (Pods) |
| Worker 2 | `k8s-w2` | `192.168.3.122` | Executa aplicações (Pods) |
| Worker 3 | `k8s-w3` | `192.168.3.123` | Executa aplicações (Pods) |
| NFS Server | `k8s-nfs` | `192.168.3.240` | Armazenamento persistente (Storage) |
| Rede | `N/A` | `192.168.3.0/24` | Sub-rede do laboratório |
| Gateway | `N/A` | `192.168.3.254` | Roteador da rede |
| MetalLB Pool | `N/A` | `192.168.3.90-98` | IPs virtuais para serviços K8s |

### Diagrama da Arquitetura

```mermaid
graph TD
    subgraph "LAN (Rede 192.168.3.0/24)"
        direction LR
        USER[💻 Usuário]
        HOST["
            <b>Hyper-V Host</b>
            <br/>(Servidor Físico)
        "]
        GW["<b>Gateway</b><br/>192.168.3.254"]

        USER -- Acesso --> LB_IP
    end

    subgraph HOST
        direction TB
        VNET["🌐 Virtual Switch Externo"]

        subgraph "Cluster Kubernetes"
            direction LR
            CP["<b>k8s-cp</b><br/>192.168.3.120"]
            W1["<b>k8S-w1</b><br/>192.168.3.121"]
            W2["<b>k8s-w2</b><br/>192.168.3.122"]
            W3["<b>k8s-w3</b><br/>192.168.3.123"]
        end

        NFS_VM["<b>k8s-nfs</b><br/>192.168.3.240"]

        VNET -- Conecta --> CP
        VNET -- Conecta --> W1
        VNET -- Conecta --> W2
        VNET -- Conecta --> W3
        VNET -- Conecta --> NFS_VM
    end

    subgraph "Lógica do Cluster"
        direction TB
        LB_IP["<b>[ MetalLB ]</b><br/>Pool: 192.168.3.90-98<br/>Load Balancer"]

        subgraph "Aplicações"
            direction LR
            APP1["<b>App: &quot;Onde estou?&quot;</b><br/>(3 réplicas)"]
            APP2["<b>App: &quot;Writers/Reader&quot;</b><br/>(4 pods)"]
        end

        subgraph "Armazenamento"
            direction TB
            PROV[<b>NFS Provisioner</b>]
            SC["<b>StorageClass 'nfs-storage'</b><br/>(ReadWriteMany)"]
        end

        LB_IP -- Roteia para --> APP1
        APP2 -- Monta PVC (RWX) --> SC
        SC -- Usa --> PROV
        PROV -- Monta (NFS) --> NFS_VM
    end
