![AWS](https://img.shields.io/badge/AWS-VPC-orange)

# aws-secure-network-project

Projeto prático implementando uma rede segura na AWS: VPC privada, EC2 sem IP público, e VPC Endpoints (Interface/Gateway) permitindo acesso autenticado via IAM através do SSM Session Manager.

## Objetivo

Demonstrar uma arquitetura de rede segura na AWS, combinando um Security Group sem regras de entrada, uma VPC sem acesso à internet pública, e três VPC Endpoints (SSM, SSMMessages e EC2Messages) para permitir acesso totalmente privado e auditável a uma instância EC2.

## Ambiente

- Amazon VPC
- Amazon EC2
- AWS Systems Manager (Session Manager)
- VPC Endpoints (Interface e Gateway)
- Security Groups

## Estrutura de Governança

```
Root
├── VPC
│   ├── Route Table Privada (sem rota para Internet Gateway)
│   ├── Security Groups
│   │   ├── Security Group da EC2 (sem regras de entrada)
│   │   └── Security Group dos Endpoints (entrada 443 apenas do SG da EC2)
│   ├── VPC Endpoints
│   │   ├── com.amazonaws.<região>.ssm
│   │   ├── com.amazonaws.<região>.ssmmessages
│   │   └── com.amazonaws.<região>.ec2messages
│   └── Subnet Privada
│       └── Instância EC2 (sem IP público, com IAM Role de permissões SSM)
```

## Fluxo da Arquitetura

1. A instância EC2 é criada em uma subnet privada, sem IP público e sem regras de entrada.
2. Uma IAM Role anexada à instância concede permissão para se comunicar com o AWS Systems Manager.
3. O tráfego da instância chega ao SSM exclusivamente através dos três VPC Endpoints do tipo Interface, usando AWS PrivateLink em vez da internet pública.
4. O AWS Systems Manager Session Manager autentica a conexão via IAM e abre uma sessão de terminal criptografada e auditável — sem chaves SSH ou portas abertas.

## Configuração Principal

- **CIDR da VPC:** 10.0.0.0/16
- **CIDR da Subnet Privada:** 10.0.1.0/24
- **Route Table:** Sem rota para Internet Gateway (totalmente privada)
- **Security Group da EC2:** Sem regras de entrada; saída HTTPS (443) permitida
- **Security Group dos Endpoints:** Entrada HTTPS (443) permitida apenas do Security Group da EC2
- **VPC Endpoints (Interface):**
  - `com.amazonaws.<região>.ssm` — API principal do Systems Manager
  - `com.amazonaws.<região>.ssmmessages` — canal de dados do Session Manager
  - `com.amazonaws.<região>.ec2messages` — comunicação entre o agente e o serviço
- **IAM Role:** Anexada à instância EC2 com a política `AmazonSSMManagedInstanceCore`

## Evidências

### 1. Criando um security group sem regras de entrada
![Security Group](images/security-group-no-inbound.png)

### 2. Criando a instância EC2 sem IP público
![EC2 Instance](images/ec2-no-public-ip.png)

### 3. Conectando à instância EC2 e testando a sessão
![Session Connected](images/session-manager-connected.png)

### 4. Criando os três VPC endpoints necessários
![VPC Endpoints](images/vpc-endpoints-created.png)

## Notas

- Apenas três Interface Endpoints são necessários para o Session Manager funcionar em uma subnet totalmente privada: `ssm`, `ssmmessages` e `ec2messages`.
- As regras do Security Group precisam ser configuradas de forma bidirecional: a instância EC2 precisa de saída na porta 443, e o Security Group dos Endpoints precisa de entrada na porta 443 restrita ao Security Group da EC2 — a ausência de qualquer um dos lados causa timeout de conexão.
- Um Gateway Endpoint de S3 também foi criado, pois o SSM Agent usa o S3 internamente para buscar atualizações. Diferente dos Interface Endpoints, os Gateway Endpoints são gratuitos e roteiam o tráfego via tabela de rotas, em vez de uma ENI.
- Essa arquitetura elimina a necessidade de bastion host, chaves SSH ou portas de entrada abertas, alinhando-se aos princípios de segurança Zero Trust e ao CIS Benchmark.
