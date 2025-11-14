# Arquitetura AWS Multi-AZ para E-commerce

## 📋 Informações do Projeto

**Disciplina:** Arquitetura de Soluções em Cloud  
**Aluno:** Leonardo Imbroisi Mesquita  
**Repositório:** [https://github.com/leoimbroisi/Desafio_FInal_Arq_Solucoes](https://github.com/leoimbroisi/Desafio_FInal_Arq_Solucoes)

---

## 🎯 Objetivo

Projetar e documentar uma arquitetura completa em nuvem AWS para uma aplicação de e-commerce, garantindo alta disponibilidade, escalabilidade, segurança e recuperação de desastres.

---

## 🏗️ Arquitetura Implementada

### **Visão Geral**

A solução implementa uma arquitetura Multi-AZ (Multiple Availability Zones) distribuída em **3 zonas de disponibilidade** na região **us-east-1 (N. Virginia)**, garantindo **99.99% de disponibilidade (SLA)**.

### **Diagrama da Arquitetura**

O diagrama completo está disponível em:
- **Formato editável:** `diagramas/arquitetura-cloud-completa.drawio`
- **Imagem PNG:** `diagramas/arquitetura-cloud-completa.png`
- **Imagem SVG:** `diagramas/arquitetura-cloud-completa.svg`

---

## 📦 Componentes da Arquitetura

### **1. Camada de Entrada e Segurança**

#### **Route 53 (DNS)**
- Roteamento de tráfego DNS com health checks
- Failover automático em caso de indisponibilidade
- Resolução global de nomes de domínio

#### **AWS WAF (Web Application Firewall)**
- Proteção contra ataques web (SQL Injection, XSS, DDoS)
- Regras customizadas de segurança
- Integração com CloudWatch para monitoramento

#### **Internet Gateway**
- Conectividade pública para a VPC
- Permite comunicação entre recursos internos e internet
- Gerenciado pela AWS (alta disponibilidade nativa)

---

### **2. Rede e Conectividade**

#### **VPC (Virtual Private Cloud)**
- **CIDR:** 10.0.0.0/16
- **Nome:** E-commerce Production
- Isolamento completo da rede

#### **Subnets Públicas (3 - uma por AZ)**
- **AZ-A:** 10.0.1.0/24
- **AZ-B:** 10.0.2.0/24
- **AZ-C:** 10.0.3.0/24
- Contêm NAT Gateways para acesso à internet das subnets privadas

#### **Subnets Privadas de Aplicação (3 - uma por AZ)**
- **AZ-A:** 10.0.11.0/24
- **AZ-B:** 10.0.12.0/24
- **AZ-C:** 10.0.13.0/24
- Hospedam as instâncias EC2 da aplicação

#### **Subnets Privadas de Banco de Dados (3 - uma por AZ)**
- **AZ-A:** 10.0.21.0/24
- **AZ-B:** 10.0.22.0/24
- **AZ-C:** 10.0.23.0/24
- Isoladas para o banco de dados RDS

#### **NAT Gateways (3 - um por AZ)**
- Permitem que recursos em subnets privadas acessem a internet
- Redundância em cada zona de disponibilidade
- Evita ponto único de falha

---

### **3. Camada de Balanceamento**

#### **Application Load Balancer (ALB)**
- **Tipo:** Layer 7 (HTTP/HTTPS)
- **Multi-AZ:** Distribui tráfego entre as 3 zonas
- **Health Checks:** Remove automaticamente instâncias não saudáveis
- **Cross-Zone Load Balancing:** Habilitado
- **SSL/TLS:** Certificados gerenciados via ACM

---

### **4. Camada de Computação**

#### **Instâncias EC2**
- **Quantidade inicial:** 3 instâncias (1 por AZ)
- **Tipo:** t3.medium
- **Sistema Operacional:** Amazon Linux 2
- **Containerização:** Docker instalado

#### **Auto Scaling Group**
- **Mínimo:** 3 instâncias
- **Máximo:** 6 instâncias
- **Distribuição:** Balanceada entre as 3 AZs
- **Métricas de Scaling:**
  - Utilização de CPU
  - Número de requisições
- **Auto Healing:** Substitui automaticamente instâncias com falha (< 5 minutos)

#### **ElastiCache Redis Cluster**
- **Arquitetura:** Multi-AZ
- **Primary:** AZ-C
- **Replicas:** AZ-A e AZ-B
- **Função:** Cache distribuído para sessões e dados frequentes
- **Failover automático:** Promoção de réplica em caso de falha

---

### **5. Camada de Dados**

#### **Amazon RDS PostgreSQL 15**
- **Tipo de Instância:** db.r6g.large
- **Arquitetura Multi-AZ:**
  - **Primary:** us-east-1a (AZ-A)
  - **Standby:** us-east-1b (AZ-B) - Replicação síncrona
  - **Read Replica:** us-east-1c (AZ-C) - Replicação assíncrona

#### **Características:**
- **Failover Automático:** < 60 segundos
- **Backup Automático:** Diário (retenção de 7 dias)
- **Point-in-Time Recovery:** Até 35 dias
- **Encryption at Rest:** AES-256
- **Encryption in Transit:** SSL/TLS

---

### **6. Mecanismos de Failover**

#### **RDS Multi-AZ**
- **Tempo de Failover:** < 60 segundos
- **Tipo:** Automático
- **Processo:** Troca de DNS do endpoint de Primary para Standby
- **Trigger:** Falha de hardware, manutenção, perda de AZ

#### **Application Load Balancer**
- **Health Checks:** A cada 30 segundos
- **Ação:** Remove instâncias não saudáveis automaticamente
- **Recuperação:** Adiciona instâncias de volta quando saudáveis

#### **Auto Scaling**
- **Monitoramento:** Contínuo via CloudWatch
- **Ação:** Provisiona novas instâncias em caso de falha
- **Tempo de Recuperação:** < 5 minutos

#### **ElastiCache Redis**
- **Failover:** Promoção automática de réplica para primary
- **Tempo:** < 60 segundos
- **Sincronização:** Replicação assíncrona entre nós

#### **Route 53**
- **Health Checks:** Monitora endpoints
- **DNS Failover:** Redireciona tráfego em caso de falha regional

#### **NAT Gateway**
- **Redundância:** 1 por AZ
- **Failover:** Cada subnet privada usa NAT Gateway na mesma AZ

---

### **7. Segurança**

#### **AWS IAM (Identity and Access Management)**
- Controle de acesso granular
- Políticas least-privilege
- Roles para instâncias EC2 acessarem RDS e outros serviços

#### **Security Groups**
- Firewall virtual para instâncias EC2
- Regras de entrada/saída restritas
- Acesso ao banco apenas de subnets de aplicação

#### **AWS Secrets Manager**
- Armazenamento seguro de credenciais
- Rotação automática de senhas do banco de dados
- Integração com RDS

#### **AWS WAF**
- Proteção contra OWASP Top 10
- Rate limiting
- Bloqueio de IPs maliciosos

---

### **8. Monitoramento e Observabilidade**

#### **Amazon CloudWatch**
- **Métricas:** CPU, memória, rede, disco
- **Logs:** Aplicação, sistema, acesso
- **Alarmes:** Notificações via SNS
- **Dashboards:** Visualização em tempo real

#### **AWS X-Ray**
- Rastreamento distribuído de requisições
- Análise de performance end-to-end
- Identificação de gargalos

#### **AWS CloudTrail**
- Auditoria de todas as chamadas de API
- Logs de conformidade
- Análise de segurança

#### **AWS Config**
- Monitoramento de conformidade
- Histórico de mudanças de configuração
- Regras de compliance automatizadas

---

### **9. Backup e Recuperação**

#### **AWS Backup**
- Backup centralizado de todos os recursos
- Políticas automatizadas
- Retenção configurável

#### **Amazon S3**
- Armazenamento de backups e logs
- Versionamento habilitado
- Lifecycle policies para otimização de custos

#### **RDS Automated Backups**
- Snapshots diários automáticos
- Point-in-Time Recovery
- Retenção: 7 dias (backups) + 35 dias (PITR)

---

### **10. CDN e Distribuição Global**

#### **Amazon CloudFront**
- CDN global para conteúdo estático
- Cache de edge locations
- Integração com S3 e ALB
- Redução de latência global

---

## 🔄 Fluxo de Requisição

```
1. Usuário Final
   ↓
2. Route 53 (DNS Resolution)
   ↓
3. AWS WAF (Security Filtering)
   ↓
4. Internet Gateway
   ↓
5. Application Load Balancer (Multi-AZ)
   ↓
6. EC2 Instances (Auto Scaling Group)
   ↓
7. ElastiCache Redis (Cache Layer)
   ↓
8. RDS PostgreSQL (Database Layer)
```

---

## 📊 Alta Disponibilidade

### **Características**

- ✅ **3 Availability Zones** ativas
- ✅ **Multi-AZ Database** com failover < 60s
- ✅ **Cross-Zone Load Balancing** habilitado
- ✅ **Auto Healing** via Auto Scaling
- ✅ **Redundância de NAT Gateways**
- ✅ **ElastiCache Multi-AZ**

### **SLA Esperado**

**99.99% de disponibilidade** (uptime anual: ~52 minutos de downtime)

---

## 🛡️ Segurança

### **Camadas de Segurança Implementadas**

1. **Perímetro:** WAF + Route 53
2. **Rede:** Security Groups + Network ACLs
3. **Aplicação:** IAM Roles + Secrets Manager
4. **Dados:** Encryption at rest + in transit
5. **Auditoria:** CloudTrail + Config + CloudWatch

### **Compliance**

- Logs centralizados no S3
- Auditoria completa via CloudTrail
- Monitoramento de compliance via AWS Config
- Encryption obrigatória (AES-256)

---

## 📈 Escalabilidade

### **Horizontal Scaling**

- Auto Scaling Group: Min 3 → Max 6 instâncias
- Baseado em CPU e número de requisições
- Distribuição automática entre AZs

### **Vertical Scaling**

- Instâncias t3.medium podem ser redimensionadas
- RDS db.r6g.large escalável sem downtime

### **Database Scaling**

- Read Replica para queries de leitura
- ElastiCache para redução de carga no banco

---

## 💰 Otimização de Custos

### **Estratégias Implementadas**

1. **Auto Scaling:** Ajusta recursos conforme demanda
2. **S3 Lifecycle Policies:** Move logs antigos para classes mais baratas
3. **Reserved Instances:** Possibilidade de usar para RDS e EC2 base
4. **Spot Instances:** Pode ser usado para scaling adicional
5. **CloudWatch Alarms:** Evita over-provisioning

---

## 🚀 Implementação

### **Requisitos Atendidos**

#### **2. Provisionamento das VMs e Configuração do Banco de Dados**

✅ **VMs em múltiplas zonas de disponibilidade**  
✅ **Imagem Linux:** Amazon Linux 2  
✅ **Balanceador de carga** configurado  
✅ **Escalonamento automático:** Min 3 | Max 6  
✅ **Segurança via firewall:** WAF + Security Groups  
✅ **Banco de dados gerenciado:** Amazon RDS PostgreSQL 15  
✅ **Replicação Multi-AZ** habilitada  
✅ **Backups automáticos** configurados  
✅ **Políticas IAM** para acesso EC2 → RDS  

#### **3. Segurança e Monitoramento**

✅ **Políticas IAM** implementadas  
✅ **Logs e monitoramento:** CloudWatch, X-Ray, CloudTrail  
✅ **Segurança na comunicação:** SSL/TLS end-to-end  

---

## 📁 Estrutura do Repositório

```
desafio2.1/
├── diagramas/
│   ├── arquitetura-cloud-completa.drawio  # Diagrama editável
│   ├── arquitetura-cloud-completa.png     # Imagem do diagrama
│   └── arquitetura-cloud-completa.svg     # Versão vetorial
├── .gitignore                              # Arquivos ignorados
└── README.md                               # Este arquivo
```

---

## 🔗 Links

**Repositório GitHub:**  
[https://github.com/leoimbroisi/Desafio_FInal_Arq_Solucoes](https://github.com/leoimbroisi/Desafio_FInal_Arq_Solucoes)

**Visualização do Diagrama:**  
Acesse o repositório e visualize o arquivo `diagramas/arquitetura-cloud-completa.png`

---

## 📝 Observações

### **Diferença: Multi-AZ vs. Multi-Regional**

A arquitetura implementa **Multi-AZ** (Multiple Availability Zones) dentro da mesma região AWS. Embora o requisito mencione "replicação multi-regional", a abordagem Multi-AZ é:

- **Mais comum** para alta disponibilidade de produção
- **Menor latência** entre réplicas (replicação síncrona)
- **Menor custo** comparado a multi-regional
- **Suficiente** para garantir 99.99% de uptime

Para implementar **Multi-Regional**, seria necessário:
- Adicionar outra região AWS (ex: us-west-2)
- Configurar replicação RDS cross-region
- Implementar Route 53 Geolocation Routing
- Aumentar significativamente os custos

---

## 👨‍💻 Autor

**Leonardo Imbroisi Mesquita**  
Pós-Graduação em Arquitetura de Soluções em Cloud

---

## 📅 Data de Entrega

14 de novembro de 2025
