# Ponderada — Provisionamento de Infraestrutura AWS com Terraform (IaC)

**Aluno:** Kauan Massuia
**Disciplina:** Módulo 8 — Prof. Romualdo
**Tutorial base:** [HashiCorp — Create Infrastructure](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-create)

---

## 📋 Sumário

1. [Objetivo](#-objetivo)
2. [Por que LocalStack ao invés da AWS?](#-por-que-localstack-ao-invés-da-aws)
3. [O que é IaC e Terraform?](#-o-que-é-iac-e-terraform)
4. [Tecnologias Utilizadas](#-tecnologias-utilizadas)
5. [Pré-requisitos](#-pré-requisitos)
6. [Estrutura do Projeto](#-estrutura-do-projeto)
7. [Passo a Passo Completo](#-passo-a-passo-completo)
8. [Recursos Provisionados](#-recursos-provisionados-na-nuvem)
9. [Destruição da Infraestrutura](#-destruição-da-infraestrutura)
10. [Código para AWS Real vs LocalStack](#-código-para-aws-real-vs-localstack)
11. [Referências](#-referências)

---

## 🎯 Objetivo

Seguir o tutorial oficial da HashiCorp **"Create Infrastructure"** para provisionar uma instância **EC2** (máquina virtual) na AWS utilizando **Terraform** como ferramenta de **Infraestrutura como Código (IaC)**.

A execução prática foi realizada com o **LocalStack** — emulador local dos serviços AWS via Docker — conforme justificado na seção abaixo.

---

## ⚠️ Por que LocalStack ao invés da AWS?

Durante a tentativa de seguir o tutorial utilizando a AWS real, enfrentei os seguintes problemas:

### Problema 1 — Conta AWS pessoal incompleta
Ao criar uma conta pessoal na AWS, o processo exige **validação de cartão de crédito internacional**. A AWS cobra uma taxa temporária de ~$1 USD para verificar o cartão. Não consegui completar essa etapa de validação, e a tela de cadastro ficou travada em `portal.aws.amazon.com/billing/signup/incomplete` com a mensagem **"Complete your account setup"**.

### Problema 2 — AWS Academy inacessível
A faculdade disponibiliza o **AWS Academy Learner Lab** (ambiente gratuito para alunos). Porém, ao tentar acessar, não consegui recuperar a senha da conta — o e-mail de redefinição simplesmente não chegava, impossibilitando o login e o acesso ao laboratório.

### Solução adotada — LocalStack
O **LocalStack** é uma ferramenta open-source amplamente utilizada no mercado de DevOps e SRE que emula os serviços da AWS localmente via Docker. Com ele:

- O fluxo do Terraform é **idêntico** ao da AWS real (mesmos comandos, mesmos outputs)
- Não é necessário cartão de crédito, conta AWS ou credenciais reais
- É usado em produção por empresas para testes de infraestrutura antes de aplicar na nuvem real
- O código Terraform é **100% compatível** — basta trocar os endpoints para migrar para a AWS real

> **Nota:** O arquivo `main_aws.tf.example` neste repositório contém a versão exata do código para uso com a AWS real, incluindo o bloco `data "aws_ami"` do tutorial original.

---

## 💡 O que é IaC e Terraform?

**Infraestrutura como Código (IaC)** é a prática de gerenciar servidores, redes e bancos de dados por meio de arquivos de configuração declarativos, ao invés de processos manuais em painéis web.

| Benefício | Descrição |
|---|---|
| **Reprodutibilidade** | O mesmo código gera o mesmo ambiente toda vez |
| **Versionamento** | A infra fica no Git com histórico de mudanças |
| **Automação** | Reduz erros humanos e acelera deploys |
| **Documentação viva** | O código *é* a documentação |

O **Terraform** da HashiCorp utiliza a linguagem **HCL** e suporta centenas de provedores (AWS, Azure, GCP). Ele funciona em 3 etapas: `init` → `plan` → `apply`.

---

## 🛠 Tecnologias Utilizadas

| Tecnologia | Versão | Propósito |
|---|---|---|
| **Terraform** | v1.9.5 | Ferramenta de IaC |
| **AWS Provider** | v5.100.0 | Plugin para APIs da AWS |
| **LocalStack** | v3.4.0 (Community) | Emulador local da AWS via Docker |
| **Docker** | - | Container runtime para o LocalStack |
| **Git/GitHub** | - | Versionamento do código |

---

## 📦 Pré-requisitos

1. **Terraform CLI** (v1.2+) — [Instalação](https://developer.hashicorp.com/terraform/install)
2. **Docker** — [Instalação](https://docs.docker.com/get-docker/)
3. **Git** — [Instalação](https://git-scm.com/)

---

## 📁 Estrutura do Projeto

```
ponderadaromualdo8/
├── .gitignore               # Ignora .tfstate, .terraform/, .env
├── .terraform.lock.hcl      # Lock file com versão exata do provider
├── terraform.tf             # Bloco terraform{} — providers e versão
├── main.tf                  # Provider AWS (LocalStack) + recurso EC2
├── main_aws.tf.example      # Versão original do tutorial (AWS real)
├── docker-compose.yml       # Compose para subir o LocalStack
└── README.md                # Documentação (este arquivo)
```

---

## 🚀 Passo a Passo Completo

### Passo 1 — Criar o diretório do projeto

Conforme o tutorial, o primeiro passo é criar um diretório dedicado para os arquivos `.tf`:

```bash
$ mkdir ponderadaromualdo8
$ cd ponderadaromualdo8
$ git init
```

> **Propósito:** O Terraform carrega automaticamente todos os arquivos `.tf` do diretório atual, por isso cada projeto deve ter seu próprio diretório.

---

### Passo 2 — Subir o LocalStack com Docker

O LocalStack disponibiliza um endpoint em `http://localhost:4566` que aceita chamadas idênticas às APIs reais da AWS.

```bash
$ docker run -d --name localstack \
    -p 4566:4566 \
    -p 4510-4559:4510-4559 \
    localstack/localstack:3.4.0
```

**Verificação — container saudável:**
```
$ docker ps

CONTAINER ID   IMAGE                         STATUS                    PORTS                              NAMES
64784d1f0543   localstack/localstack:3.4.0   Up (healthy)              0.0.0.0:4566->4566/tcp, ...        localstack
```

> **Propósito:** Simular os serviços EC2, S3, IAM e STS da AWS sem custos.

---

### Passo 3 — Criar o arquivo `terraform.tf` (Bloco Terraform)

O bloco `terraform {}` define quais **providers** (plugins) o Terraform deve baixar e qual versão mínima é necessária.

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

**O que cada campo significa:**
- `source = "hashicorp/aws"` → Endereço do provider no [Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws)
- `version = "~> 5.92"` → Aceita versões `>= 5.92` e `< 6.0` (major version travada)
- `required_version = ">= 1.2"` → Qualquer Terraform >= 1.2 pode executar

**Verificação da versão instalada:**
```
$ terraform -version

Terraform v1.9.5
on linux_amd64
```

---

### Passo 4 — Criar o arquivo `main.tf` (Provider + Recurso)

Este arquivo define o **provider** (como se conectar à AWS) e o **resource** (o que criar).

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

**Explicação dos blocos:**

| Bloco | O que faz |
|---|---|
| `provider "aws"` | Configura a conexão com a AWS. O `endpoints` redireciona para o LocalStack |
| `access_key / secret_key = "test"` | Credenciais dummy aceitas pelo LocalStack |
| `skip_credentials_validation` | Desativa a validação de credenciais (necessário para LocalStack) |
| `resource "aws_instance"` | Define a instância EC2 a ser criada |
| `ami = "ami-0026a04369a3093cc"` | AMI do Ubuntu 24.04 LTS (Noble Numbat) |
| `instance_type = "t2.micro"` | 1 vCPU, 1 GiB RAM — elegível ao AWS Free Tier |
| `tags.Name = "learn-terraform"` | Nome da instância no console |

> **Nota sobre o `data "aws_ami"` do tutorial original:**
> O tutorial da HashiCorp usa um bloco `data "aws_ami" "ubuntu"` que busca dinamicamente a AMI mais recente do Ubuntu. O LocalStack Community Edition não suporta o filtro de AMIs, por isso usamos o ID da AMI diretamente (`ami-0026a04369a3093cc`), que é o mesmo ID retornado pelo tutorial original. O arquivo `main_aws.tf.example` contém a versão com o `data` block para uso na AWS real.

---

### Passo 5 — Formatar a configuração (`terraform fmt`)

O `terraform fmt` reformata todos os arquivos `.tf` segundo o estilo oficial da HashiCorp.

```
$ terraform fmt
```

> Se nenhum arquivo for listado na saída, significa que todos já estavam formatados corretamente.

---

### Passo 6 — Inicializar o workspace (`terraform init`)

O `terraform init` baixa e instala os plugins dos providers definidos no `terraform.tf`.

```
$ terraform init

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

**O que aconteceu:**
- O Terraform baixou o provider AWS **v5.100.0** e armazenou em `.terraform/`
- Criou o arquivo `.terraform.lock.hcl` que trava a versão exata do provider

---

### Passo 7 — Validar a configuração (`terraform validate`)

O `terraform validate` verifica se a sintaxe HCL está correta e se não há referências inválidas.

```
$ terraform validate

Success! The configuration is valid.
```

---

### Passo 8 — Criar a infraestrutura (`terraform apply`)

O `terraform apply` primeiro gera um **plano de execução** mostrando o que será criado, e depois aplica as mudanças.

```
$ terraform apply -auto-approve

Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_instance.app_server will be created
  + resource "aws_instance" "app_server" {
      + ami                                  = "ami-0026a04369a3093cc"
      + arn                                  = (known after apply)
      + associate_public_ip_address          = (known after apply)
      + availability_zone                    = (known after apply)
      + disable_api_stop                     = (known after apply)
      + disable_api_termination              = (known after apply)
      + ebs_optimized                        = (known after apply)
      + get_password_data                    = false
      + id                                   = (known after apply)
      + instance_state                       = (known after apply)
      + instance_type                        = "t2.micro"
      + ipv6_address_count                   = (known after apply)
      + key_name                             = (known after apply)
      + monitoring                           = (known after apply)
      + primary_network_interface_id         = (known after apply)
      + private_dns                          = (known after apply)
      + private_ip                           = (known after apply)
      + public_dns                           = (known after apply)
      + public_ip                            = (known after apply)
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
aws_instance.app_server: Creating...
aws_instance.app_server: Still creating... [10s elapsed]
aws_instance.app_server: Creation complete after 11s [id=i-ea5b989e35804262b]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

**Resultado:** ✅ Instância EC2 criada com ID `i-ea5b989e35804262b`

**Leitura do output:**
- O símbolo `+` indica que o recurso será **criado**
- Valores `(known after apply)` são definidos pela AWS/LocalStack após a criação
- `Plan: 1 to add` confirma que exatamente 1 recurso será criado
- `Creation complete after 11s` confirma o provisionamento bem-sucedido

---

### Passo 9 — Inspecionar o estado (`terraform state list` / `terraform show`)

O Terraform armazena o estado da infraestrutura no arquivo `terraform.tfstate`. Podemos inspecioná-lo:

**Listar recursos:**
```
$ terraform state list

aws_instance.app_server
```

**Exibir detalhes completos do estado:**
```
$ terraform show

# aws_instance.app_server:
resource "aws_instance" "app_server" {
    ami                                  = "ami-0026a04369a3093cc"
    arn                                  = "arn:aws:ec2:us-east-1::instance/i-ea5b989e35804262b"
    associate_public_ip_address          = true
    availability_zone                    = "us-east-1a"
    disable_api_stop                     = false
    disable_api_termination              = false
    ebs_optimized                        = false
    get_password_data                    = false
    id                                   = "i-ea5b989e35804262b"
    instance_initiated_shutdown_behavior = "stop"
    instance_state                       = "running"
    instance_type                        = "t2.micro"
    ipv6_address_count                   = 0
    monitoring                           = false
    primary_network_interface_id         = "eni-a37878d4"
    private_dns                          = "ip-10-236-201-159.ec2.internal"
    private_ip                           = "10.236.201.159"
    public_dns                           = "ec2-54-214-40-81.compute-1.amazonaws.com"
    public_ip                            = "54.214.40.81"
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
        volume_id             = "vol-969cceef"
        volume_size           = 8
        volume_type           = "gp2"
    }
}
```

**O que o `terraform show` confirma:**
- A instância está em estado **`running`** ✅
- Recebeu IP privado (`10.236.201.159`) e público (`54.214.40.81`)
- Foi criada na AZ `us-east-1a` com a tag `Name = "learn-terraform"`
- Possui um volume EBS root de 8 GB tipo `gp2`

---

## ☁️ Recursos Provisionados na Nuvem

### Instância EC2 (`aws_instance.app_server`)

| Propriedade | Valor |
|---|---|
| **Instance ID** | `i-ea5b989e35804262b` |
| **AMI** | `ami-0026a04369a3093cc` (Ubuntu 24.04 LTS Noble Numbat) |
| **Tipo** | `t2.micro` (1 vCPU, 1 GiB RAM — Free Tier) |
| **Estado** | `running` ✅ |
| **Região** | `us-east-1` |
| **Availability Zone** | `us-east-1a` |
| **IP Privado** | `10.236.201.159` |
| **IP Público** | `54.214.40.81` |
| **DNS Privado** | `ip-10-236-201-159.ec2.internal` |
| **DNS Público** | `ec2-54-214-40-81.compute-1.amazonaws.com` |
| **Subnet** | `subnet-572b3fbd` |
| **ARN** | `arn:aws:ec2:us-east-1::instance/i-ea5b989e35804262b` |

### Volume EBS (Root Block Device)

| Propriedade | Valor |
|---|---|
| **Volume ID** | `vol-969cceef` |
| **Device** | `/dev/sda1` |
| **Tamanho** | 8 GB |
| **Tipo** | `gp2` (General Purpose SSD) |
| **Criptografado** | Não |
| **Delete on Termination** | Sim |

### Interface de Rede

| Propriedade | Valor |
|---|---|
| **ENI ID** | `eni-a37878d4` |
| **IP Público associado** | Sim (`54.214.40.81`) |

---

## 💣 Destruição da Infraestrutura

O `terraform destroy` remove todos os recursos gerenciados. Esta é uma etapa **obrigatória** em ambientes AWS reais para evitar cobranças.

```
$ terraform destroy -auto-approve

aws_instance.app_server: Refreshing state... [id=i-1fbb843bf40238d8c]

Terraform will perform the following actions:

  # aws_instance.app_server will be destroyed
  - resource "aws_instance" "app_server" {
      - ami               = "ami-0026a04369a3093cc" -> null
      - id                = "i-1fbb843bf40238d8c" -> null
      - instance_state    = "running" -> null
      - instance_type     = "t2.micro" -> null
      - tags              = {
          - "Name" = "learn-terraform"
        } -> null
      ...
    }

Plan: 0 to add, 0 to change, 1 to destroy.
aws_instance.app_server: Destroying... [id=i-1fbb843bf40238d8c]
aws_instance.app_server: Still destroying... [10s elapsed]
aws_instance.app_server: Destruction complete after 10s

Destroy complete! Resources: 1 destroyed.
```

Para também parar o LocalStack:
```bash
$ docker stop localstack && docker rm localstack
```

---

## 🔀 Código para AWS Real vs LocalStack

O repositório contém dois arquivos de configuração para comparação:

### `main.tf` — Versão LocalStack (usada neste projeto)
```hcl
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
  tags = { Name = "learn-terraform" }
}
```

### `main_aws.tf.example` — Versão AWS Real (tutorial original)
```hcl
provider "aws" {
  region = "us-west-2"
}

data "aws_ami" "ubuntu" {
  most_recent = true
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }
  owners = ["099720109477"] # Canonical
}

resource "aws_instance" "app_server" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  tags = { Name = "learn-terraform" }
}
```

**Diferenças principais:**

| Aspecto | LocalStack | AWS Real |
|---|---|---|
| **Endpoints** | `http://localhost:4566` | APIs oficiais da AWS |
| **Credenciais** | `test`/`test` (dummy) | `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` reais |
| **AMI** | ID hardcoded | `data "aws_ami"` busca dinamicamente |
| **Região** | `us-east-1` | `us-west-2` (ou qualquer outra) |
| **Custo** | Gratuito | Free Tier (t2.micro grátis por 12 meses) |

> O bloco `data "aws_ami"` do tutorial original não é suportado pelo LocalStack Community, por isso usamos o ID da AMI diretamente. Em produção, o `data` block é a prática recomendada pois mantém a configuração dinâmica.

---

## 📚 Referências

- [Tutorial HashiCorp — Create Infrastructure](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-create)
- [Documentação do Terraform](https://developer.hashicorp.com/terraform/docs)
- [AWS Provider — Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [LocalStack — GitHub](https://github.com/localstack/localstack)
- [Docker — Get Started](https://docs.docker.com/get-started/)