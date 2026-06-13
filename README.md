# Ponderada — Provisionamento de Infraestrutura AWS com Terraform (IaC)

**Aluno:** Kauan Massuia  
**Disciplina:** Módulo 8 — Prof. Romualdo  
**Data:** Junho de 2026

---

## 📋 Sumário

1. [Objetivo](#-objetivo)
2. [O que é Infraestrutura como Código (IaC)?](#-o-que-é-infraestrutura-como-código-iac)
3. [Tecnologias Utilizadas](#-tecnologias-utilizadas)
4. [Pré-requisitos](#-pré-requisitos)
5. [Estrutura do Projeto](#-estrutura-do-projeto)
6. [Passo a Passo da Execução](#-passo-a-passo-da-execução)
   - [Passo 1 — Criar o diretório do projeto](#passo-1--criar-o-diretório-do-projeto)
   - [Passo 2 — Subir o ambiente AWS local com LocalStack](#passo-2--subir-o-ambiente-aws-local-com-localstack)
   - [Passo 3 — Criar o arquivo terraform.tf](#passo-3--criar-o-arquivo-terraformtf)
   - [Passo 4 — Criar o arquivo main.tf](#passo-4--criar-o-arquivo-maintf)
   - [Passo 5 — Formatar a configuração (terraform fmt)](#passo-5--formatar-a-configuração-terraform-fmt)
   - [Passo 6 — Inicializar o workspace (terraform init)](#passo-6--inicializar-o-workspace-terraform-init)
   - [Passo 7 — Validar a configuração (terraform validate)](#passo-7--validar-a-configuração-terraform-validate)
   - [Passo 8 — Gerar o plano de execução (terraform plan)](#passo-8--gerar-o-plano-de-execução-terraform-plan)
   - [Passo 9 — Aplicar a infraestrutura (terraform apply)](#passo-9--aplicar-a-infraestrutura-terraform-apply)
   - [Passo 10 — Inspecionar o estado (terraform state / terraform show)](#passo-10--inspecionar-o-estado-terraform-state--terraform-show)
7. [Recursos Provisionados na Nuvem](#-recursos-provisionados-na-nuvem)
8. [Destruição da Infraestrutura](#-destruição-da-infraestrutura)
9. [Referências](#-referências)

---

## 🎯 Objetivo

Seguir o tutorial oficial da HashiCorp [**"Create infrastructure"**](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-create) para provisionar uma instância **EC2** na AWS utilizando **Terraform** como ferramenta de Infraestrutura como Código (IaC).

Para a execução prática, utilizei o **LocalStack** — uma ferramenta que emula os serviços da AWS localmente via Docker, permitindo rodar o fluxo completo do Terraform sem necessidade de uma conta AWS ativa ou cartão de crédito.

---

## 💡 O que é Infraestrutura como Código (IaC)?

**Infraestrutura como Código (IaC)** é a prática de gerenciar e provisionar recursos de infraestrutura (servidores, redes, bancos de dados) por meio de arquivos de configuração declarativos, ao invés de processos manuais em painéis web.

**Benefícios principais:**
- **Reprodutibilidade:** o mesmo código gera o mesmo ambiente toda vez.
- **Versionamento:** a infraestrutura fica versionada no Git, com histórico de mudanças.
- **Automação:** reduz erros humanos e acelera deploys.
- **Documentação viva:** o código *é* a documentação da infra.

O **Terraform** da HashiCorp é uma das ferramentas de IaC mais populares do mercado. Ele utiliza a linguagem **HCL (HashiCorp Configuration Language)** e suporta centenas de provedores de nuvem (AWS, Azure, GCP, etc.).

---

## 🛠 Tecnologias Utilizadas

| Tecnologia | Versão | Propósito |
|---|---|---|
| **Terraform** | v1.9.5 | Ferramenta de IaC para provisionar recursos |
| **AWS Provider** | v5.100.0 | Plugin do Terraform para interagir com APIs da AWS |
| **LocalStack** | v3.4.0 (Community) | Emulador local dos serviços AWS via Docker |
| **Docker** | - | Runtime de containers para rodar o LocalStack |
| **Git/GitHub** | - | Versionamento e hospedagem do código |

---

## 📦 Pré-requisitos

Para reproduzir este projeto, é necessário ter instalado:

1. **Terraform CLI** (v1.2+) — [Instalação oficial](https://developer.hashicorp.com/terraform/install)
2. **Docker** — [Instalação oficial](https://docs.docker.com/get-docker/)
3. **Git** — [Instalação oficial](https://git-scm.com/)

---

## 📁 Estrutura do Projeto

```
ponderadaromualdo8/
├── .gitignore              # Ignora arquivos sensíveis e temporários do Terraform
├── .terraform.lock.hcl     # Lock file com versões exatas dos providers
├── terraform.tf            # Bloco terraform{} com required_providers e versão
├── main.tf                 # Provider AWS + recurso EC2 (instância)
└── README.md               # Este arquivo de documentação
```

---

## 🚀 Passo a Passo da Execução

### Passo 1 — Criar o diretório do projeto

Primeiro, criei o diretório para armazenar os arquivos de configuração do Terraform e inicializei o repositório Git.

```bash
$ mkdir ponderadaromualdo8
$ cd ponderadaromualdo8
$ git init
```

**Propósito:** organizar os arquivos `.tf` em um diretório dedicado que também serve como repositório Git.

---

### Passo 2 — Subir o ambiente AWS local com LocalStack

O **LocalStack** emula os serviços da AWS localmente. Ele roda como um container Docker e disponibiliza um endpoint (`http://localhost:4566`) que aceita chamadas idênticas às APIs reais da AWS.

```bash
$ docker run -d --name localstack \
    -p 4566:4566 \
    -p 4510-4559:4510-4559 \
    localstack/localstack:3.4.0
```

Após subir, verifiquei a saúde do container:

```bash
$ docker ps
```

```
CONTAINER ID   IMAGE                         STATUS                    PORTS                              NAMES
64784d1f0543   localstack/localstack:3.4.0   Up About a minute (healthy)   0.0.0.0:4566->4566/tcp, ...   localstack
```

**Propósito:** fornecer um ambiente que simula os serviços da AWS (EC2, S3, IAM, etc.) sem custos, sem cartão de crédito e sem credenciais reais.

---

### Passo 3 — Criar o arquivo `terraform.tf`

Este arquivo define a configuração do próprio Terraform: quais **providers** usar e qual versão mínima do Terraform é necessária.

```hcl
# terraform.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.92"
    }
  }

  required_version = ">= 1.2"
}
```

**Propósito:**
- `required_providers` → define que usaremos o provider `hashicorp/aws` com versão `>= 5.92` e `< 6.0`.
- `required_version` → garante que qualquer versão do Terraform >= 1.2 pode executar este projeto.

---

### Passo 4 — Criar o arquivo `main.tf`

Este arquivo configura o **provider AWS** (apontando para o LocalStack) e define o **recurso EC2** que será provisionado.

```hcl
# main.tf

provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true

  endpoints {
    ec2 = "http://localhost:4566"
    iam = "http://localhost:4566"
    s3  = "http://localhost:4566"
    sts = "http://localhost:4566"
  }
}

resource "aws_instance" "app_server" {
  ami           = "ami-0026a04369a3093cc"
  instance_type = "t2.micro"

  tags = {
    Name = "learn-terraform"
  }
}
```

**Propósito:**
- **Provider block** → configura a região (`us-east-1`), credenciais dummy (`test`/`test`) e redireciona as chamadas de API para o LocalStack (`http://localhost:4566`).
- **Resource block** → define uma instância EC2 com AMI Ubuntu (`ami-0026a04369a3093cc`), tipo `t2.micro` (elegível ao Free Tier da AWS) e a tag `Name = "learn-terraform"`.

> **Nota:** Em um ambiente AWS real, o bloco `provider` seria simplesmente:
> ```hcl
> provider "aws" {
>   region = "us-west-2"
> }
> ```
> E as credenciais seriam fornecidas via variáveis de ambiente (`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`).

---

### Passo 5 — Formatar a configuração (`terraform fmt`)

O comando `terraform fmt` reformata automaticamente os arquivos `.tf` de acordo com o estilo recomendado pela HashiCorp.

```bash
$ terraform fmt
```

**Propósito:** garantir formatação consistente e legível em todos os arquivos de configuração.

---

### Passo 6 — Inicializar o workspace (`terraform init`)

O `terraform init` baixa e instala os plugins dos providers definidos no `terraform.tf`.

```bash
$ terraform init
```

**Resultado obtido:**
```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.92"...
- Installing hashicorp/aws v5.100.0...
- Installed hashicorp/aws v5.100.0 (signed by HashiCorp)
Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

**Propósito:** inicializar o workspace, baixar o provider AWS (`v5.100.0`) e criar o arquivo de lock (`.terraform.lock.hcl`) que trava as versões exatas para garantir consistência.

---

### Passo 7 — Validar a configuração (`terraform validate`)

O `terraform validate` verifica se a sintaxe HCL está correta e se a configuração é internamente consistente.

```bash
$ terraform validate
```

**Resultado obtido:**
```
Success! The configuration is valid.
```

**Propósito:** identificar erros de sintaxe ou referências inválidas antes de tentar provisionar recursos.

---

### Passo 8 — Gerar o plano de execução (`terraform plan`)

O `terraform plan` analisa a configuração e mostra exatamente quais recursos serão criados, modificados ou destruídos, **sem fazer nenhuma alteração real**.

```bash
$ terraform plan
```

**Resultado obtido:**
```
Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_instance.app_server will be created
  + resource "aws_instance" "app_server" {
      + ami                                  = "ami-0026a04369a3093cc"
      + arn                                  = (known after apply)
      + associate_public_ip_address          = (known after apply)
      + availability_zone                    = (known after apply)
      + cpu_core_count                       = (known after apply)
      + cpu_threads_per_core                 = (known after apply)
      + disable_api_stop                     = (known after apply)
      + disable_api_termination              = (known after apply)
      + ebs_optimized                        = (known after apply)
      + get_password_data                    = false
      + id                                   = (known after apply)
      + instance_initiated_shutdown_behavior = (known after apply)
      + instance_state                       = (known after apply)
      + instance_type                        = "t2.micro"
      + ipv6_address_count                   = (known after apply)
      + ipv6_addresses                       = (known after apply)
      + key_name                             = (known after apply)
      + monitoring                           = (known after apply)
      + primary_network_interface_id         = (known after apply)
      + private_dns                          = (known after apply)
      + private_ip                           = (known after apply)
      + public_dns                           = (known after apply)
      + public_ip                            = (known after apply)
      + secondary_private_ips                = (known after apply)
      + security_groups                      = (known after apply)
      + source_dest_check                    = true
      + subnet_id                            = (known after apply)
      + tags                                 = {
          + "Name" = "learn-terraform"
        }
      + tags_all                             = {
          + "Name" = "learn-terraform"
        }
      + tenancy                              = (known after apply)
      + user_data_replace_on_change          = false
      + vpc_security_group_ids               = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

**Propósito:** revisar as mudanças propostas antes de aplicá-las. O símbolo `+` indica que o recurso será **criado**. Valores marcados como `(known after apply)` só serão definidos após a criação efetiva do recurso.

---

### Passo 9 — Aplicar a infraestrutura (`terraform apply`)

O `terraform apply` executa o plano e efetivamente provisiona os recursos na nuvem (ou no LocalStack, no nosso caso).

```bash
$ terraform apply -auto-approve
```

**Resultado obtido:**
```
Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_instance.app_server will be created
  + resource "aws_instance" "app_server" {
      + ami                                  = "ami-0026a04369a3093cc"
      + instance_type                        = "t2.micro"
      + tags                                 = {
          + "Name" = "learn-terraform"
        }
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
aws_instance.app_server: Creating...
aws_instance.app_server: Still creating... [10s elapsed]
aws_instance.app_server: Creation complete after 13s [id=i-1fbb843bf40238d8c]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

✅ **Instância EC2 criada com sucesso!** ID: `i-1fbb843bf40238d8c`

**Propósito:** provisionar efetivamente a infraestrutura definida nos arquivos `.tf`. O Terraform criou a instância EC2 e registrou o estado no arquivo `terraform.tfstate`.

---

### Passo 10 — Inspecionar o estado (`terraform state` / `terraform show`)

Após a aplicação, o Terraform armazena dados sobre a infraestrutura criada no arquivo `terraform.tfstate`. Podemos inspecionar o estado com os comandos abaixo.

**Listar recursos no estado:**
```bash
$ terraform state list
```

```
aws_instance.app_server
```

**Exibir detalhes completos:**
```bash
$ terraform show
```

```
# aws_instance.app_server:
resource "aws_instance" "app_server" {
    ami                                  = "ami-0026a04369a3093cc"
    arn                                  = "arn:aws:ec2:us-east-1::instance/i-1fbb843bf40238d8c"
    associate_public_ip_address          = true
    availability_zone                    = "us-east-1a"
    disable_api_stop                     = false
    disable_api_termination              = false
    ebs_optimized                        = false
    get_password_data                    = false
    id                                   = "i-1fbb843bf40238d8c"
    instance_initiated_shutdown_behavior = "stop"
    instance_state                       = "running"
    instance_type                        = "t2.micro"
    ipv6_address_count                   = 0
    monitoring                           = false
    primary_network_interface_id         = "eni-38a34a67"
    private_dns                          = "ip-10-101-104-61.ec2.internal"
    private_ip                           = "10.101.104.61"
    public_dns                           = "ec2-54-214-127-161.compute-1.amazonaws.com"
    public_ip                            = "54.214.127.161"
    source_dest_check                    = true
    subnet_id                            = "subnet-572b3fbd"
    tags                                 = {
        "Name" = "learn-terraform"
    }
    tags_all                             = {
        "Name" = "learn-terraform"
    }
    tenancy                              = "default"
    user_data_replace_on_change          = false

    root_block_device {
        delete_on_termination = true
        device_name           = "/dev/sda1"
        encrypted             = false
        volume_id             = "vol-571518ad"
        volume_size           = 8
        volume_type           = "gp2"
    }
}
```

**Propósito:** verificar o estado atual da infraestrutura gerenciada pelo Terraform. O `terraform show` exibe todas as propriedades do recurso, confirmando que a instância está em estado `running`.

---

## ☁️ Recursos Provisionados na Nuvem

A tabela abaixo documenta todos os recursos que foram provisionados pelo Terraform via LocalStack:

### Instância EC2

| Propriedade | Valor |
|---|---|
| **Instance ID** | `i-1fbb843bf40238d8c` |
| **AMI** | `ami-0026a04369a3093cc` (Ubuntu 24.04 LTS) |
| **Tipo da instância** | `t2.micro` (1 vCPU, 1 GiB RAM) |
| **Estado** | `running` ✅ |
| **Região** | `us-east-1` |
| **Availability Zone** | `us-east-1a` |
| **IP Privado** | `10.101.104.61` |
| **IP Público** | `54.214.127.161` |
| **DNS Privado** | `ip-10-101-104-61.ec2.internal` |
| **DNS Público** | `ec2-54-214-127-161.compute-1.amazonaws.com` |
| **Subnet** | `subnet-572b3fbd` |
| **Tag Name** | `learn-terraform` |

### Volume EBS (Root Block Device)

| Propriedade | Valor |
|---|---|
| **Volume ID** | `vol-571518ad` |
| **Device** | `/dev/sda1` |
| **Tamanho** | 8 GB |
| **Tipo** | `gp2` |
| **Criptografado** | Não |
| **Delete on Termination** | Sim |

### Interface de Rede

| Propriedade | Valor |
|---|---|
| **ENI ID** | `eni-38a34a67` |
| **IP Público associado** | Sim |

---

## 💣 Destruição da Infraestrutura

Para destruir todos os recursos provisionados e evitar custos, utiliza-se o comando:

```bash
$ terraform destroy -auto-approve
```

No nosso caso, como usamos o LocalStack (emulação local), basta parar e remover o container Docker:

```bash
$ docker stop localstack
$ docker rm localstack
```

**Propósito:** garantir que nenhum recurso fique ativo gerando custos. Em um ambiente AWS real, o `terraform destroy` é **fundamental** para encerrar recursos após testes.

---

## 📚 Referências

- [Tutorial oficial HashiCorp — Create Infrastructure](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-create)
- [Documentação do Terraform](https://developer.hashicorp.com/terraform/docs)
- [Documentação do AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [LocalStack — Emulador AWS Local](https://github.com/localstack/localstack)
- [Docker — Get Started](https://docs.docker.com/get-started/)