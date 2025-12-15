# Documentação Completa - Projeto Sorteador Strigus

## 📋 Índice

1. [Visão Geral do Projeto](#visão-geral-do-projeto)
2. [Arquitetura do Sistema](#arquitetura-do-sistema)
3. [Estrutura dos Repositórios](#estrutura-dos-repositórios)
4. [Componentes Detalhados](#componentes-detalhados)
5. [Fluxo CI/CD Completo](#fluxo-cicd-completo)
6. [Infraestrutura como Código](#infraestrutura-como-código)
7. [Configuração e Deploy](#configuração-e-deploy)
8. [Segurança](#segurança)
9. [Monitoramento e Observabilidade](#monitoramento-e-observabilidade)
10. [Troubleshooting](#troubleshooting)

---

## 🎯 Visão Geral do Projeto

O **Sorteador Strigus** é um projeto completo de DevOps que implementa uma aplicação web para realização de sorteios em lives da LinuxTips. O projeto é dividido em três repositórios separados, cada um com responsabilidades específicas:

1. **Sorteador-Strigus** - Aplicação principal (frontend + backend)
2. **linuxtips-cicd-reusable** - Pipelines de CI/CD reutilizáveis
3. **linuxtips-terraform-eks** - Infraestrutura como código (IaC)

### Objetivo da Aplicação

A aplicação permite:
- Carregar arquivos CSV/XLSX com participantes
- Realizar sorteios aleatórios
- Modo Live com interação em tempo real
- Persistência de dados em Redis
- Interface web moderna e responsiva

---

## 🏗️ Arquitetura do Sistema

### Diagrama de Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│                        GitHub Actions                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Workflow Principal (Sorteador-Strigus)                  │  │
│  │  ┌──────────────┐         ┌──────────────┐               │  │
│  │  │  CI Pipeline │────────▶│  CD Pipeline │               │  │
│  │  │  (reusable)  │         │  (reusable)  │               │  │
│  │  └──────────────┘         └──────────────┘               │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                         AWS Cloud                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Amazon ECR (Container Registry)                         │  │
│  │  • Imagens Docker assinadas com Cosign                    │  │
│  │  • Scan automático de vulnerabilidades                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            │                                     │
│                            ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Amazon EKS (Kubernetes Cluster)                         │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │  Istio Service Mesh                                │  │  │
│  │  │  ┌──────────────┐  ┌──────────────┐               │  │  │
│  │  │  │  Gateway     │  │  VirtualService│              │  │  │
│  │  │  └──────────────┘  └──────────────┘               │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │  ArgoCD (GitOps)                                   │  │  │
│  │  │  • Sincronização automática de manifests           │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │  Sorteador-Strigus Deployment                      │  │  │
│  │  │  • Pods com Redis + Flask                          │  │  │
│  │  │  • Service (ClusterIP)                             │  │  │
│  │  │  • Health checks (startup/readiness/liveness)      │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            │                                     │
│                            ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Network Load Balancer (NLB)                             │  │
│  │  • SSL/TLS termination                                   │  │
│  │  • Roteamento para Istio Gateway                        │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │   Internet    │
                    │  (Usuários)   │
                    └───────────────┘
```

### Fluxo de Dados

1. **Desenvolvimento**: Código commitado no repositório Sorteador-Strigus
2. **CI**: GitHub Actions executa pipeline de CI (build, scan, assinatura)
3. **Registry**: Imagem publicada no ECR com assinatura Cosign
4. **CD**: Pipeline de CD verifica assinatura e atualiza manifests
5. **GitOps**: ArgoCD detecta mudanças e faz deploy no EKS
6. **Aplicação**: Pods executam a aplicação com Redis embutido
7. **Roteamento**: NLB → Istio Gateway → VirtualService → Service → Pods

---

## 📁 Estrutura dos Repositórios

### 1. Sorteador-Strigus (Aplicação)

**Localização**: `/home/paulo/Sorteador-Strigus`

**Responsabilidades**:
- Código-fonte da aplicação (Python Flask + JavaScript)
- Dockerfile para containerização
- Manifests Helm para Kubernetes
- Configurações de Istio (Gateway, VirtualService)
- Workflow principal do GitHub Actions

**Estrutura de Diretórios**:
```
Sorteador-Strigus/
├── app/                    # Código da aplicação
│   ├── app.py             # Backend Flask
│   ├── Dockerfile         # Imagem container
│   ├── requirements.txt   # Dependências Python
│   ├── go-bin-exec/       # Binário Go para Redis+Flask
│   ├── static/            # Arquivos estáticos (CSS, JS)
│   └── templates/         # Templates HTML
├── manifests/             # Helm Chart
│   ├── Chart.yaml         # Metadados do chart
│   ├── values.yaml        # Valores configuráveis
│   └── templates/         # Templates Kubernetes
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── istio-gateway.yaml
│       └── istio-virtualservice.yaml
├── k8s/                   # Manifests Kubernetes (alternativo)
├── .github/
│   └── workflows/
│       └── main.yml       # Workflow principal (chama CI/CD)
└── README.md              # Documentação da aplicação
```

**Tecnologias**:
- **Backend**: Python 3, Flask, Pandas, NumPy
- **Frontend**: JavaScript, HTML5, CSS3
- **Cache**: Redis (embutido no container)
- **Container**: Distroless image com Go binary
- **Orquestração**: Kubernetes + Helm
- **Service Mesh**: Istio

### 2. linuxtips-cicd-reusable (CI/CD)

**Localização**: `/home/paulo/linuxtips-cicd-reusable`

**Responsabilidades**:
- Workflows reutilizáveis de CI/CD
- Padronização de pipelines
- Integração com AWS (ECR, ECS, EKS)
- Segurança (scan, assinatura)

**Estrutura de Diretórios**:
```
linuxtips-cicd-reusable/
├── .github/
│   └── workflows/
│       ├── pipeline-ci.yaml    # Pipeline de CI
│       └── pipeline-cd.yaml    # Pipeline de CD
└── README.md                   # Documentação dos workflows
```

**Workflows Disponíveis**:

#### `pipeline-ci.yaml` - Continuous Integration
- Lint de Dockerfile (Hadolint)
- Build de imagem Docker
- Scan de vulnerabilidades (Trivy)
- Push para ECR
- Assinatura com Cosign

#### `pipeline-cd.yaml` - Continuous Delivery
- Verificação de assinatura (Cosign)
- Deploy para ECS (opcional)
- Deploy para EKS via kubectl (opcional)
- Deploy para EKS via GitOps/ArgoCD (opcional)

### 3. linuxtips-terraform-eks (Infraestrutura)

**Localização**: `/home/paulo/linuxtips-terraform-eks`

**Responsabilidades**:
- Provisionamento de infraestrutura AWS
- Cluster EKS completo
- Addons e ferramentas (Istio, ArgoCD, etc.)
- Configuração de segurança e acesso
- Network Load Balancer

**Estrutura de Diretórios**:
```
linuxtips-terraform-eks/
└── iac/                      # Infrastructure as Code
    ├── providers.tf          # Configuração do provider
    ├── variables.tf          # Variáveis do projeto
    ├── data.tf               # Data sources (VPC, subnets)
    ├── eks.tf                # Cluster EKS
    ├── node-group.tf         # Node groups
    ├── nodes_launch_template.tf
    ├── ecr.tf                # Container registry
    ├── addons.tf             # EKS Addons (CNI, CoreDNS, etc.)
    ├── oidc.tf               # OIDC provider
    ├── access_entries.tf     # Acesso ao cluster
    ├── iam_cluster.tf        # IAM roles do cluster
    ├── iam_nodes.tf          # IAM roles dos nodes
    ├── iam_aws_load_balancer_controller.tf
    ├── kms.tf                # Criptografia
    ├── security.tf           # Security groups
    ├── nlb.tf                # Network Load Balancer
    ├── helm_istio_base.tf    # Istio base
    ├── helm_istio_public.tf  # Istio ingress gateway
    ├── helm_argocd.tf        # ArgoCD
    ├── istio_argocd.tf       # Configuração Istio para ArgoCD
    ├── outputs.tf            # Outputs do Terraform
    └── files/                # Arquivos de configuração
        └── argocd-values.yml
```

**Recursos Criados**:
- **EKS Cluster**: Kubernetes 1.33
- **Node Groups**: Instâncias EC2 (c7i-flex.large)
- **ECR Repository**: Registry para imagens Docker
- **Istio Service Mesh**: Gateway e controle de tráfego
- **ArgoCD**: GitOps para deploy automático
- **Network Load Balancer**: Balanceador de carga
- **KMS**: Criptografia de secrets
- **IAM Roles**: Acesso seguro via OIDC

---

## 🔧 Componentes Detalhados

### Aplicação Sorteador-Strigus

#### Backend (Flask)

**Arquivo**: `app/app.py`

**Funcionalidades**:
- Endpoints REST para upload de arquivos
- Processamento de CSV/XLSX com Pandas
- Sorteios aleatórios com NumPy
- Integração com Redis para cache
- Health checks (`/healthz`, `/readyz`)

**Dependências** (requirements.txt):
- Flask
- Pandas
- NumPy
- Redis (via go-bin-exec)

#### Frontend

**Tecnologias**:
- JavaScript vanilla
- HTML5
- CSS3
- Contador regressivo animado

**Funcionalidades**:
- Upload de arquivos (drag & drop)
- Seleção de coluna para sorteio
- Modo Live com timer
- Interface responsiva

#### Containerização

**Dockerfile**: `app/Dockerfile`

**Características**:
- Imagem distroless (segurança)
- Binário Go executa Redis + Flask
- Multi-stage build
- Otimizado para produção

**Estrutura do Container**:
```
Container Distroless
├── Redis (via go-bin-exec)
└── Flask App (via go-bin-exec)
```

### Manifests Kubernetes (Helm)

#### Chart Helm

**Chart.yaml**:
```yaml
apiVersion: v2
name: sorteador
description: Helm chart para o app Sorteador
type: application
version: 0.1.0
appVersion: "1.0.0"
```

#### Values.yaml

**Configurações Principais**:
- **image**: Tag da imagem Docker
- **replicaCount**: Número de réplicas (padrão: 2)
- **resources**: CPU/Memory requests e limits
- **probes**: Health checks configuráveis
- **service**: Configuração do Service Kubernetes
- **istio**: Habilitar/desabilitar Istio

**Exemplo**:
```yaml
image: 870461445219.dkr.ecr.us-east-1.amazonaws.com/linuxtips/sorteador-strigus:sha
replicaCount: 2
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

#### Templates Kubernetes

**1. Deployment** (`deployment.yaml`):
- Replicas configuráveis
- Health probes (startup, readiness, liveness)
- Resource limits
- Labels padronizados

**2. Service** (`service.yaml`):
- ClusterIP (interno)
- Porta 80 → 5000 (container)
- Selector baseado em labels

**3. Istio Gateway** (`istio-gateway.yaml`):
- Porta 80 (HTTP)
- Host configurável
- Selector para Istio Ingress Gateway

**4. Istio VirtualService** (`istio-virtualservice.yaml`):
- Roteamento para o Service
- Headers de proxy (x-forwarded-proto, x-forwarded-port)
- Configuração de HTTPS

### Pipelines CI/CD

#### Pipeline CI (`pipeline-ci.yaml`)

**Etapas Detalhadas**:

1. **Checkout**: Baixa o código do repositório
2. **Dockerfile Lint**: Valida boas práticas (Hadolint)
3. **AWS OIDC**: Autenticação sem chaves
4. **ECR Login**: Autentica no registry
5. **Docker Buildx**: Setup para builds otimizados
6. **Compilar Nome**: Monta tag da imagem (`registry/imagem:sha`)
7. **Build**: Constrói imagem localmente
8. **Trivy Scan**: Escaneia vulnerabilidades (HIGH/CRITICAL)
9. **Push ECR**: Publica imagem (se scan passou)
10. **Cosign Sign**: Assina imagem com chave privada

**Condições**:
- Push só acontece se Trivy passar
- Assinatura só acontece se push passar
- Relatórios salvos como artifacts

#### Pipeline CD (`pipeline-cd.yaml`)

**Jobs Disponíveis**:

**1. prepare**:
- Compila nome da imagem
- Verifica assinatura com Cosign
- Output para jobs de deploy

**2. ecs-deploy** (se `deploy-type == 'ecs'`):
- Obtém Task Definition atual
- Atualiza com nova imagem
- Deploy no ECS
- Aguarda estabilização

**3. eks-deploy** (se `deploy-type == 'eks-kubectl'`):
- Configura kubectl
- Renderiza manifests Kubernetes
- Aplica no cluster
- Aguarda rollout

**4. eks-deploy-gitops** (se `deploy-type == 'eks-argocd'`):
- Atualiza `values.yaml` com nova imagem
- Commit e push da alteração
- ArgoCD detecta e faz deploy automaticamente

### Infraestrutura Terraform

#### Cluster EKS

**Características**:
- Versão Kubernetes: 1.33
- Encryption at rest (KMS)
- Logs habilitados (API, audit, etc.)
- Zonal shift habilitado
- OIDC provider configurado

**Configuração**:
```hcl
resource "aws_eks_cluster" "main" {
  name     = "linuxtips-eks-cluster"
  version  = "1.33"
  
  encryption_config {
    provider {
      key_arn = aws_kms_key.main.arn
    }
    resources = ["secrets"]
  }
  
  enabled_cluster_log_types = [
    "api", "audit", "authenticator", 
    "controllerManager", "scheduler"
  ]
}
```

#### Node Groups

**Configuração**:
- Tipo de instância: `c7i-flex.large`
- Desired: 2 nodes
- Min: 1 node
- Max: 2 nodes
- AMI customizada
- Subnets privadas

#### ECR Repository

**Configuração**:
```hcl
resource "aws_ecr_repository" "app" {
  name                 = "linuxtips/sorteador-strigus"
  image_tag_mutability = "MUTABLE"
  
  image_scanning_configuration {
    scan_on_push = true
  }
}
```

#### Istio Service Mesh

**Componentes**:
- **Istio Base**: Componentes core
- **Istio Ingress Gateway**: Gateway público
- **Auto-scaling**: Baseado em CPU (threshold: 80%)
- **Min replicas**: 1

#### ArgoCD

**Configuração**:
- Namespace: `argocd`
- Service type: ClusterIP
- Extensions habilitadas
- Rollout extension instalada
- Integração com Istio

#### Network Load Balancer

**Características**:
- SSL/TLS termination
- Certificado ACM
- Target: Istio Ingress Gateway
- Health checks configurados

#### Acesso e Segurança

**Access Entries**:
- Nodes (EC2_LINUX)
- GitHub Actions Role (STANDARD)
- Service User (opcional)

**IAM Policies**:
- Cluster Admin para GitHub Actions
- Permissões para ECR
- Permissões para EKS

**KMS**:
- Criptografia de secrets
- Key rotation habilitado

---

## 🔄 Fluxo CI/CD Completo

### Visão Geral

```
┌─────────────────────────────────────────────────────────────┐
│  1. Desenvolvedor faz commit/push                            │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. GitHub Actions: Workflow Principal (main.yml)           │
│     Trigger: workflow_dispatch (manual)                     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  3. JOB: ci                                                 │
│     Workflow: pipeline-ci.yaml (reusable)                   │
│     ┌─────────────────────────────────────────────────────┐ │
│     │ 3.1. Checkout código                                │ │
│     │ 3.2. Lint Dockerfile (Hadolint)                     │ │
│     │ 3.3. Autenticação AWS (OIDC)                        │ │
│     │ 3.4. Login ECR                                      │ │
│     │ 3.5. Build imagem Docker                            │ │
│     │ 3.6. Scan vulnerabilidades (Trivy)                  │ │
│     │ 3.7. Push para ECR (se scan OK)                     │ │
│     │ 3.8. Assinar imagem (Cosign)                        │ │
│     └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  4. Imagem no ECR                                           │
│     Tag: registry/imagem:commit-sha                         │
│     Assinatura: Cosign signature                            │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  5. JOB: cd (needs: ci)                                     │
│     Workflow: pipeline-cd.yaml (reusable)                   │
│     ┌─────────────────────────────────────────────────────┐ │
│     │ 5.1. Verificar assinatura (Cosign)                  │ │
│     │ 5.2. Deploy baseado em deploy-type:                │ │
│     │      • eks-argocd: Atualiza values.yaml e commit    │ │
│     │      • eks-kubectl: Aplica manifests diretamente    │ │
│     │      • ecs: Atualiza Task Definition                │ │
│     └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  6. Deploy no Kubernetes (EKS)                              │
│     ┌─────────────────────────────────────────────────────┐ │
│     │ 6.1. ArgoCD detecta mudança (GitOps)               │ │
│     │ 6.2. Sincroniza aplicação                          │ │
│     │ 6.3. Helm install/upgrade                           │ │
│     │ 6.4. Pods criados com nova imagem                  │ │
│     │ 6.5. Health checks validam                          │ │
│     │ 6.6. Rollout completo                               │ │
│     └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  7. Aplicação em Produção                                   │
│     Acessível via: https://sorteador.grupobamaq.com.br     │
└─────────────────────────────────────────────────────────────┘
```

### Detalhamento por Etapa

#### Etapa 1: Trigger

**Workflow**: `.github/workflows/main.yml` (Sorteador-Strigus)

```yaml
on:
  workflow_dispatch:  # Execução manual
```

**Como executar**:
1. Acesse GitHub Actions no repositório
2. Selecione "Pipeline CI/CD DevOps"
3. Clique em "Run workflow"

#### Etapa 2-3: CI Pipeline

**Workflow Reutilizável**: `linuxtips-cicd-reusable/.github/workflows/pipeline-ci.yaml`

**Inputs**:
- `working-directory`: `app`
- `aws-region`: `us-east-1`

**Secrets**:
- `AWS_ROLE_TO_ASSUME`: ARN da role IAM
- `COSIGN_KEY`: Chave privada (Base64)
- `COSIGN_PASSWORD`: Senha da chave

**Variáveis**:
- `IMAGE_NAME`: `linuxtips/sorteador-strigus`

**Resultado**:
- Imagem no ECR: `870461445219.dkr.ecr.us-east-1.amazonaws.com/linuxtips/sorteador-strigus:sha`
- Assinatura Cosign aplicada
- Relatórios de segurança (artifacts)

#### Etapa 4-5: CD Pipeline

**Workflow Reutilizável**: `linuxtips-cicd-reusable/.github/workflows/pipeline-cd.yaml`

**Inputs**:
- `working-directory`: `app`
- `aws-region`: `us-east-1`
- `deploy-type`: `eks-argocd` (via variável `TYPE`)

**Secrets**:
- `AWS_ROLE_TO_ASSUME`: ARN da role IAM
- `COSIGN_KEY_PUB`: Chave pública (Base64)
- `COSIGN_PASSWORD`: Senha da chave

**Variáveis**:
- `IMAGE_NAME`: `linuxtips/sorteador-strigus`
- `K8S_NAMESPACE`: Namespace Kubernetes
- `EKS_CLUSTER_NAME`: Nome do cluster

**Processo GitOps**:
1. Verifica assinatura da imagem
2. Atualiza `manifests/values.yaml` com nova tag
3. Commit e push da alteração
4. ArgoCD detecta mudança automaticamente

#### Etapa 6: Deploy no Kubernetes

**ArgoCD**:
- Monitora repositório Git
- Detecta mudança em `values.yaml`
- Executa `helm upgrade`
- Aplica novos manifests

**Kubernetes**:
- Cria novos pods com nova imagem
- Health checks validam pods
- Service roteia tráfego
- Istio Gateway expõe externamente

#### Etapa 7: Aplicação Disponível

**Roteamento**:
```
Internet
  ↓
Network Load Balancer (NLB)
  ↓
Istio Ingress Gateway
  ↓
Istio VirtualService
  ↓
Kubernetes Service (ClusterIP)
  ↓
Pods (Sorteador-Strigus)
```

---

## 🏗️ Infraestrutura como Código

### Terraform

#### Pré-requisitos

1. **Terraform**: Versão >= 1.0
2. **AWS CLI**: Configurado com credenciais
3. **kubectl**: Para acesso ao cluster
4. **Helm**: Para instalar charts

#### Variáveis Principais

**Arquivo**: `iac/variables.tf`

**Variáveis Importantes**:
```hcl
variable "aws_region" {
  default = "us-east-1"
}

variable "project_name" {
  default = "linuxtips-eks"
}

variable "cluster_version" {
  default = "1.33"
}

variable "node_instance_types" {
  default = ["c7i-flex.large"]
}

variable "node_desired_size" {
  default = 2
}
```

#### Deploy da Infraestrutura

**1. Inicializar Terraform**:
```bash
cd /home/paulo/linuxtips-terraform-eks/iac
terraform init
```

**2. Planejar Mudanças**:
```bash
terraform plan
```

**3. Aplicar Infraestrutura**:
```bash
terraform apply
```

**4. Configurar kubectl**:
```bash
aws eks update-kubeconfig \
  --name linuxtips-eks-cluster \
  --region us-east-1
```

**5. Verificar Cluster**:
```bash
kubectl get nodes
kubectl get pods -A
```

#### Recursos Criados

**EKS**:
- Cluster: `linuxtips-eks-cluster`
- Node Group: 2 instâncias EC2
- Addons: CNI, CoreDNS, Kube Proxy, Pod Identity, EBS CSI, EFS CSI, S3 CSI

**ECR**:
- Repository: `linuxtips/sorteador-strigus`
- Scan automático habilitado

**Istio**:
- Base components
- Ingress Gateway (público)
- Auto-scaling configurado

**ArgoCD**:
- Namespace: `argocd`
- Service: ClusterIP
- Extensions habilitadas

**Network Load Balancer**:
- SSL/TLS termination
- Target: Istio Gateway
- Health checks

**Segurança**:
- KMS key para criptografia
- IAM roles e policies
- OIDC provider
- Access entries

---

## ⚙️ Configuração e Deploy

### Configuração Inicial

#### 1. Configurar GitHub Secrets

**No repositório Sorteador-Strigus**:

**Secrets** (Settings → Secrets and variables → Actions → Secrets):
- `AWS_ROLE_TO_ASSUME`: ARN da role IAM para OIDC
- `COSIGN_KEY`: Chave privada em Base64
- `COSIGN_KEY_PUB`: Chave pública em Base64
- `COSIGN_PASSWORD`: Senha da chave Cosign

**Variáveis** (Settings → Secrets and variables → Actions → Variables):
- `IMAGE_NAME`: `linuxtips/sorteador-strigus`
- `TYPE`: `eks-argocd`
- `K8S_NAMESPACE`: `default` (ou outro namespace)
- `EKS_CLUSTER_NAME`: `linuxtips-eks-cluster`

#### 2. Gerar Chaves Cosign

```bash
# Instalar Cosign
wget https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
chmod +x cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign

# Gerar par de chaves
cosign generate-key-pair

# Isso cria:
# - cosign.key (privada - NUNCA compartilhar)
# - cosign.pub (pública)

# Codificar em Base64 para GitHub Secrets
cat cosign.key | base64 -w 0    # Para COSIGN_KEY
cat cosign.pub | base64 -w 0    # Para COSIGN_KEY_PUB
```

#### 3. Configurar IAM Role para OIDC

**Pré-requisitos**:
- OIDC provider configurado no AWS IAM
- Trust policy permitindo GitHub Actions

**Trust Policy Exemplo**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:OWNER/REPO:*"
        }
      }
    }
  ]
}
```

**Permissões Necessárias**:
- ECR: `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`, `ecr:PutImage`
- EKS: `eks:DescribeCluster`, `eks:UpdateKubeconfig`
- IAM: `iam:PassRole` (para roles do EKS)

### Deploy da Aplicação

#### Opção 1: Via GitHub Actions (Recomendado)

1. **Fazer commit das alterações**:
```bash
git add .
git commit -m "feat: nova funcionalidade"
git push origin main
```

2. **Executar workflow manualmente**:
   - Acesse GitHub Actions
   - Selecione "Pipeline CI/CD DevOps"
   - Clique em "Run workflow"

3. **Acompanhar execução**:
   - Job `ci`: Build, scan, assinatura
   - Job `cd`: Verificação e deploy

#### Opção 2: Deploy Manual via Helm

**1. Atualizar values.yaml**:
```yaml
image: 870461445219.dkr.ecr.us-east-1.amazonaws.com/linuxtips/sorteador-strigus:TAG
```

**2. Instalar/Atualizar via Helm**:
```bash
# Adicionar repositório (se necessário)
helm repo add <repo> <url>

# Instalar
helm install sorteador ./manifests \
  --namespace default \
  --create-namespace

# Ou atualizar
helm upgrade sorteador ./manifests \
  --namespace default
```

#### Opção 3: Deploy via ArgoCD

**1. Criar Application no ArgoCD**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sorteador-strigus
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/OWNER/Sorteador-Strigus
    targetRevision: main
    path: manifests
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**2. Aplicar via kubectl**:
```bash
kubectl apply -f argocd-application.yaml
```

**3. Verificar no ArgoCD UI**:
```bash
# Port-forward para ArgoCD
kubectl port-forward svc/argo-cd-argocd-server -n argocd 8080:443

# Acessar: https://localhost:8080
# Usuário: admin
# Senha: (obter via kubectl)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### Verificação do Deploy

**1. Verificar Pods**:
```bash
kubectl get pods -n default
kubectl describe pod <pod-name> -n default
```

**2. Verificar Service**:
```bash
kubectl get svc -n default
kubectl describe svc sorteador-service -n default
```

**3. Verificar Istio Gateway**:
```bash
kubectl get gateway -n default
kubectl get virtualservice -n default
```

**4. Verificar Logs**:
```bash
kubectl logs -f deployment/sorteador-deployment -n default
```

**5. Testar Aplicação**:
```bash
# Port-forward local
kubectl port-forward svc/sorteador-service -n default 5000:80

# Acessar: http://localhost:5000
```

---

## 🔒 Segurança

### Segurança da Imagem

#### 1. Scan de Vulnerabilidades (Trivy)

**Configuração**:
- Escaneia apenas vulnerabilidades HIGH e CRITICAL
- Ignora vulnerabilidades sem patch (`ignore-unfixed: true`)
- Falha o pipeline se encontrar vulnerabilidades

**Relatórios**:
- Salvos como artifacts no GitHub Actions
- Retenção: 30 dias
- Formato: texto (trivy-report.txt)

#### 2. Assinatura de Imagens (Cosign)

**Processo**:
- Imagem assinada após push para ECR
- Assinatura armazenada no registry
- Verificação obrigatória antes do deploy

**Benefícios**:
- Integridade: garante que a imagem não foi alterada
- Autenticidade: confirma origem da imagem
- Rastreabilidade: histórico de assinaturas

#### 3. Dockerfile Lint (Hadolint)

**Validações**:
- Boas práticas de Dockerfile
- Segurança de camadas
- Otimização de builds
- Relatório salvo como artifact

### Segurança da Infraestrutura

#### 1. Criptografia

**EKS Secrets**:
- Criptografia at rest via KMS
- Key rotation habilitado
- Resources: secrets

**ECR**:
- Criptografia de imagens
- Scan automático no push

#### 2. Autenticação e Autorização

**OIDC**:
- Autenticação sem chaves de acesso
- GitHub Actions usa OIDC
- Trust policies restritivas

**IAM**:
- Least privilege principle
- Roles específicas por função
- Policies granulares

**EKS Access**:
- Access entries (novo modelo)
- Cluster admin apenas para GitHub Actions
- Nodes com permissões mínimas

#### 3. Network Security

**Security Groups**:
- Regras restritivas
- Apenas portas necessárias
- Comunicação entre componentes isolada

**Istio**:
- Service mesh para segurança
- mTLS entre serviços (se configurado)
- Políticas de rede

### Segurança da Aplicação

#### 1. Health Checks

**Probes Configuradas**:
- **Startup**: Valida inicialização
- **Readiness**: Valida prontidão para receber tráfego
- **Liveness**: Valida que aplicação está viva

**Configuração**:
```yaml
startup:
  enabled: true
  path: /healthz
  periodSeconds: 5
  failureThreshold: 30

readiness:
  path: /readyz
  periodSeconds: 10
  failureThreshold: 6

liveness:
  path: /healthz
  initialDelaySeconds: 20
  periodSeconds: 10
  failureThreshold: 3
```

#### 2. Resource Limits

**Configuração**:
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

**Benefícios**:
- Previne consumo excessivo de recursos
- Permite melhor planejamento de capacidade
- Protege outros pods no cluster

#### 3. Distroless Image

**Características**:
- Sem shell, sem package manager
- Apenas binários necessários
- Superfície de ataque reduzida
- Menor tamanho da imagem

---

## 📊 Monitoramento e Observabilidade

### Logs do Cluster

**EKS Logs Habilitados**:
- API server
- Audit
- Authenticator
- Controller Manager
- Scheduler

**Acesso aos Logs**:
```bash
# Via CloudWatch
aws logs describe-log-groups --log-group-name-prefix /aws/eks/linuxtips-eks-cluster

# Via kubectl
kubectl logs -f deployment/sorteador-deployment -n default
```

### Health Checks

**Endpoints**:
- `/healthz`: Health check geral
- `/readyz`: Readiness check

**Monitoramento**:
- Kubernetes executa probes automaticamente
- Falhas registradas em eventos
- Pods reiniciados se liveness falhar

### Métricas

**Kubernetes Metrics**:
```bash
# CPU e Memory dos pods
kubectl top pods -n default

# CPU e Memory dos nodes
kubectl top nodes
```

**Istio Metrics** (se Prometheus configurado):
- Request rate
- Error rate
- Latency
- Throughput

### ArgoCD

**Monitoramento**:
- Status de sincronização
- Health status das aplicações
- Histórico de deploys
- Notificações de mudanças

**Acesso**:
```bash
# Port-forward
kubectl port-forward svc/argo-cd-argocd-server -n argocd 8080:443

# UI: https://localhost:8080
```

---

## 🔧 Troubleshooting

### Problemas Comuns

#### 1. Pipeline CI Falha

**Problema**: Trivy encontra vulnerabilidades

**Solução**:
- Revisar relatório de vulnerabilidades
- Atualizar base image ou dependências
- Aplicar patches de segurança
- Se necessário, adicionar exceções (não recomendado)

**Problema**: Falha no build da imagem

**Solução**:
- Verificar Dockerfile
- Verificar dependências (requirements.txt)
- Verificar contexto de build
- Revisar logs do GitHub Actions

#### 2. Pipeline CD Falha

**Problema**: Verificação de assinatura falha

**Solução**:
- Verificar se chave pública está correta
- Verificar se imagem foi assinada no CI
- Verificar formato da chave (Base64)
- Verificar permissões no ECR

**Problema**: Deploy no EKS falha

**Solução**:
- Verificar credenciais AWS (OIDC)
- Verificar acesso ao cluster
- Verificar manifests Kubernetes
- Verificar recursos disponíveis (CPU/Memory)

#### 3. Aplicação Não Inicia

**Problema**: Pods em CrashLoopBackOff

**Solução**:
```bash
# Verificar logs
kubectl logs <pod-name> -n default

# Verificar eventos
kubectl describe pod <pod-name> -n default

# Verificar health checks
kubectl get pods -n default -o wide
```

**Problema**: Health checks falhando

**Solução**:
- Verificar se endpoints `/healthz` e `/readyz` estão funcionando
- Verificar configuração de probes
- Verificar portas do container
- Verificar conectividade de rede

#### 4. Aplicação Não Acessível

**Problema**: Não consegue acessar via URL

**Solução**:
```bash
# Verificar Service
kubectl get svc -n default

# Verificar Istio Gateway
kubectl get gateway -n default
kubectl describe gateway sorteador-gateway -n default

# Verificar VirtualService
kubectl get virtualservice -n default
kubectl describe virtualservice sorteador-virtualservice -n default

# Verificar NLB
aws elbv2 describe-load-balancers --region us-east-1

# Verificar DNS
nslookup sorteador.grupobamaq.com.br
```

#### 5. ArgoCD Não Sincroniza

**Problema**: Aplicação não atualiza automaticamente

**Solução**:
```bash
# Verificar status da aplicação
kubectl get application -n argocd

# Verificar logs do ArgoCD
kubectl logs -f deployment/argo-cd-argocd-application-controller -n argocd

# Forçar sincronização
argocd app sync sorteador-strigus

# Verificar permissões Git
# Verificar configuração do repositório
```

### Comandos Úteis

**Kubernetes**:
```bash
# Listar recursos
kubectl get all -n default

# Descrever recurso
kubectl describe deployment sorteador-deployment -n default

# Editar recurso
kubectl edit deployment sorteador-deployment -n default

# Deletar recurso
kubectl delete deployment sorteador-deployment -n default

# Executar comando no pod
kubectl exec -it <pod-name> -n default -- /bin/sh

# Port-forward
kubectl port-forward svc/sorteador-service -n default 5000:80
```

**Terraform**:
```bash
# Validar configuração
terraform validate

# Formatar código
terraform fmt

# Ver outputs
terraform output

# Destruir infraestrutura (cuidado!)
terraform destroy
```

**AWS**:
```bash
# Listar imagens no ECR
aws ecr list-images --repository-name linuxtips/sorteador-strigus --region us-east-1

# Descrever cluster
aws eks describe-cluster --name linuxtips-eks-cluster --region us-east-1

# Verificar logs
aws logs tail /aws/eks/linuxtips-eks-cluster/api --follow --region us-east-1
```

---

## 📚 Referências e Recursos

### Documentação Oficial

- **Kubernetes**: https://kubernetes.io/docs/
- **Helm**: https://helm.sh/docs/
- **Istio**: https://istio.io/latest/docs/
- **ArgoCD**: https://argo-cd.readthedocs.io/
- **Terraform**: https://www.terraform.io/docs
- **AWS EKS**: https://docs.aws.amazon.com/eks/
- **GitHub Actions**: https://docs.github.com/en/actions
- **Cosign**: https://github.com/sigstore/cosign
- **Trivy**: https://github.com/aquasecurity/trivy
- **Hadolint**: https://github.com/hadolint/hadolint

### Repositórios do Projeto

- **Aplicação**: `Sorteador-Strigus`
- **CI/CD**: `linuxtips-cicd-reusable`
- **Infraestrutura**: `linuxtips-terraform-eks`

### Boas Práticas

- **12-Factor App**: https://12factor.net/
- **Kubernetes Best Practices**: https://kubernetes.io/docs/concepts/configuration/overview/
- **Container Security**: https://kubernetes.io/docs/concepts/security/pod-security-standards/
- **GitOps**: https://www.gitops.tech/

---

## 📝 Conclusão

Este projeto demonstra uma implementação completa de DevOps moderna, incluindo:

✅ **Infraestrutura como Código** (Terraform)
✅ **CI/CD Automatizado** (GitHub Actions)
✅ **Containerização** (Docker)
✅ **Orquestração** (Kubernetes/EKS)
✅ **Service Mesh** (Istio)
✅ **GitOps** (ArgoCD)
✅ **Segurança** (Scan, Assinatura, Criptografia)
✅ **Observabilidade** (Logs, Health Checks)

A separação em três repositórios permite:
- **Reutilização**: Workflows CI/CD podem ser usados em outros projetos
- **Manutenibilidade**: Cada componente tem responsabilidade clara
- **Escalabilidade**: Fácil adicionar novas aplicações
- **Colaboração**: Equipes podem trabalhar independentemente

---

**Última atualização**: 2024
**Versão da documentação**: 1.0.0

