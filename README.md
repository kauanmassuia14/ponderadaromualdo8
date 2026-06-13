# Ponderada 8 — Terraform + AWS (IaC)

**Kauan Massuia** — Módulo 8, Prof. Romualdo

Essa ponderada segue o tutorial [Create Infrastructure](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-create) da HashiCorp, onde a gente provisiona uma instância EC2 na AWS usando Terraform como Infraestrutura como Código.

Como não consegui usar a AWS diretamente (explico mais abaixo), rodei tudo localmente com o **LocalStack**, que simula os serviços da AWS no Docker. O código Terraform é o mesmo — só muda o endpoint.

---

## Sumário

1. [O que rolou com a AWS](#o-que-rolou-com-a-aws)
2. [Ferramentas usadas](#ferramentas-usadas)
3. [Estrutura do repositório](#estrutura-do-repositório)
4. [Passo a passo](#passo-a-passo)
5. [Recursos provisionados](#recursos-provisionados)
6. [Destruindo a infra](#destruindo-a-infra)
7. [Diferença entre o código local e o da AWS real](#diferença-entre-o-código-local-e-o-da-aws-real)
8. [Referências](#referências)

---

## O que rolou com a AWS

Antes de mais nada, preciso explicar por que estou usando o LocalStack ao invés da AWS de verdade. Tentei de duas formas e não deu certo em nenhuma:

**Tentativa 1 — Criar conta pessoal na AWS:**
Fui criar uma conta na AWS do zero, mas na hora de validar o cadastro a AWS pede um cartão de crédito internacional. Fiz o cadastro inteiro, mas a verificação do cartão não completou e a página ficou travada em `portal.aws.amazon.com/billing/signup/incomplete` com a mensagem "Complete your account setup". Sem a validação do cartão, a conta fica bloqueada e não dá pra usar nenhum serviço.

**Tentativa 2 — AWS Academy da faculdade:**
A faculdade disponibiliza o AWS Academy Learner Lab, que é um ambiente gratuito pra alunos. O problema é que eu esqueci a senha e tentei redefinir, mas o e-mail de recuperação simplesmente nunca chegou. Tentei várias vezes e nada.

**A solução — LocalStack:**
O LocalStack é uma ferramenta open-source que roda no Docker e simula os serviços da AWS localmente. É bastante usado no mercado pra testes de infraestrutura antes de aplicar na nuvem de verdade. O fluxo do Terraform funciona igualzinho — mesmos comandos, mesmas saídas. A única diferença é que as chamadas de API vão pra `localhost:4566` ao invés dos servidores da AWS.

No repositório tem o arquivo `main_aws.tf.example` com o código original do tutorial pra quem quiser rodar na AWS real.

---

## Ferramentas usadas

| Ferramenta | Versão | Pra quê |
|---|---|---|
| Terraform | v1.9.5 | Provisionar a infra via código |
| AWS Provider | v5.100.0 | Plugin que conecta o Terraform nas APIs da AWS |
| LocalStack | v3.4.0 | Emular a AWS localmente |
| Docker | — | Rodar o container do LocalStack |
| Git/GitHub | — | Versionar o código |

---

## Estrutura do repositório

```
ponderadaromualdo8/
├── .gitignore               # Ignora arquivos sensíveis (.tfstate, .env)
├── .terraform.lock.hcl      # Versão exata do provider instalado
├── terraform.tf             # Configuração do Terraform (providers, versão)
├── main.tf                  # Provider AWS + recurso EC2 (versão LocalStack)
├── main_aws.tf.example      # Versão original do tutorial pra AWS real
├── docker-compose.yml       # Compose pra subir o LocalStack
└── README.md                # Documentação (esse arquivo)
```

---

## Passo a passo

### 1. Criar o diretório do projeto

O Terraform carrega todos os `.tf` do diretório atual automaticamente, então cada projeto precisa do seu próprio diretório.

```bash
$ mkdir ponderadaromualdo8
$ cd ponderadaromualdo8
$ git init
```

---

### 2. Subir o LocalStack no Docker

O LocalStack sobe um endpoint em `localhost:4566` que responde igualzinho às APIs da AWS.

```bash
$ docker run -d --name localstack \
    -p 4566:4566 \
    -p 4510-4559:4510-4559 \
    localstack/localstack:3.4.0
```

Pra conferir se subiu certo:
```
$ docker ps

CONTAINER ID   IMAGE                         STATUS                    PORTS                              NAMES
64784d1f0543   localstack/localstack:3.4.0   Up (healthy)              0.0.0.0:4566->4566/tcp, ...        localstack
```

O `(healthy)` mostra que tá tudo rodando.

---

### 3. Criar o `terraform.tf`

Esse arquivo configura o Terraform em si — qual provider usar e qual versão mínima aceitar.

```hcl
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

- `source = "hashicorp/aws"` → busca o provider da AWS no [Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws)
- `version = "~> 5.92"` → aceita qualquer versão 5.x a partir da 5.92
- `required_version = ">= 1.2"` → precisa do Terraform 1.2 ou mais novo

Pra conferir a versão instalada:
```
$ terraform -version

Terraform v1.9.5
on linux_amd64
```

---

### 4. Criar o `main.tf`

Aqui é onde a mágica acontece. Esse arquivo define o **provider** (como se conectar) e o **resource** (o que criar).

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

  tags = {
    Name = "learn-terraform"
  }
}
```

Sobre cada parte:
- O bloco `provider` aponta pro LocalStack (`localhost:4566`) com credenciais dummy (`test`/`test`)
- O `skip_credentials_validation` e os outros `skip_` são necessários porque o LocalStack não valida credenciais como a AWS real faz
- O `resource "aws_instance"` cria a instância EC2 com a AMI do Ubuntu 24.04 e tipo `t2.micro` (que seria Free Tier na AWS real)
- A tag `Name = "learn-terraform"` é o nome que aparece no console

**Sobre o bloco `data "aws_ami"` do tutorial original:** o tutorial da HashiCorp usa um `data "aws_ami"` que busca a AMI mais recente do Ubuntu automaticamente. O LocalStack Community não suporta esse filtro, então coloquei o ID da AMI direto (`ami-0026a04369a3093cc` — que é exatamente o ID que o tutorial retorna). O arquivo `main_aws.tf.example` tem a versão com o `data` block pra quem for usar na AWS de verdade.

---

### 5. Formatar o código (`terraform fmt`)

Reformata os arquivos `.tf` pro padrão da HashiCorp.

```
$ terraform fmt
```

Se não imprimiu nada, é porque já tava formatado certo.

---

### 6. Inicializar o workspace (`terraform init`)

Esse é o primeiro comando que a gente roda. Ele baixa o provider da AWS e prepara o diretório.

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

O Terraform baixou o provider `hashicorp/aws v5.100.0`, guardou na pasta `.terraform/` e criou o `.terraform.lock.hcl` que trava a versão exata pra garantir que todo mundo use a mesma.

---

### 7. Validar a configuração (`terraform validate`)

Checa se a sintaxe dos arquivos `.tf` tá correta.

```
$ terraform validate

Success! The configuration is valid.
```

---

### 8. Provisionar a instância (`terraform apply`)

Esse é o comando que de fato cria os recursos. Ele primeiro mostra o plano (o que vai fazer) e depois aplica.

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

O `+` do lado de cada atributo significa que vai ser criado. Campos com `(known after apply)` são valores que a AWS/LocalStack define na hora da criação (como IP, ID, etc). O resultado final: **instância criada com sucesso**, ID `i-ea5b989e35804262b`.

---

### 9. Inspecionar o estado (`terraform state list` / `terraform show`)

Depois do apply, o Terraform salva tudo que criou num arquivo chamado `terraform.tfstate`. Dá pra consultar o que tem lá:

```
$ terraform state list

aws_instance.app_server
```

E pra ver os detalhes completos:

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

A instância tá com `instance_state = "running"` — rodando certinho. Recebeu IP privado, IP público, DNS, subnet e tudo mais.

---

## Recursos provisionados

Tudo que o Terraform criou, extraído diretamente do `terraform show`:

### Instância EC2

| Campo | Valor |
|---|---|
| Instance ID | `i-ea5b989e35804262b` |
| AMI | `ami-0026a04369a3093cc` (Ubuntu 24.04 LTS) |
| Tipo | `t2.micro` (1 vCPU, 1 GiB RAM) |
| Estado | `running` |
| Região / AZ | `us-east-1` / `us-east-1a` |
| IP Privado | `10.236.201.159` |
| IP Público | `54.214.40.81` |
| DNS Privado | `ip-10-236-201-159.ec2.internal` |
| DNS Público | `ec2-54-214-40-81.compute-1.amazonaws.com` |
| Subnet | `subnet-572b3fbd` |
| ARN | `arn:aws:ec2:us-east-1::instance/i-ea5b989e35804262b` |
| Tag Name | `learn-terraform` |

### Volume EBS (disco)

| Campo | Valor |
|---|---|
| Volume ID | `vol-969cceef` |
| Device | `/dev/sda1` |
| Tamanho | 8 GB |
| Tipo | `gp2` (SSD) |
| Criptografado | Não |
| Apaga junto com a instância | Sim |

### Interface de rede

| Campo | Valor |
|---|---|
| ENI ID | `eni-a37878d4` |
| Tem IP público | Sim |

---

## Destruindo a infra

Pra destruir tudo que foi criado (na AWS real isso evita cobranças):

```
$ terraform destroy -auto-approve

aws_instance.app_server: Refreshing state... [id=i-1fbb843bf40238d8c]

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

O `-` ao lado de cada atributo mostra que tá sendo **removido**. No final: `1 destroyed`.

Pra parar o LocalStack também:
```bash
$ docker stop localstack && docker rm localstack
```

---

## Diferença entre o código local e o da AWS real

O repositório tem dois arquivos pra comparar:

**`main.tf`** (o que usei — aponta pro LocalStack):
```hcl
provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  # ... endpoints apontando pra localhost:4566
}

resource "aws_instance" "app_server" {
  ami           = "ami-0026a04369a3093cc"   # AMI fixa
  instance_type = "t2.micro"
  tags = { Name = "learn-terraform" }
}
```

**`main_aws.tf.example`** (versão original do tutorial pra AWS real):
```hcl
provider "aws" {
  region = "us-west-2"
}

data "aws_ami" "ubuntu" {        # Busca a AMI mais recente automaticamente
  most_recent = true
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }
  owners = ["099720109477"]      # Canonical
}

resource "aws_instance" "app_server" {
  ami           = data.aws_ami.ubuntu.id   # AMI dinâmica
  instance_type = "t2.micro"
  tags = { Name = "learn-terraform" }
}
```

As diferenças na prática:

| | LocalStack | AWS Real |
|---|---|---|
| Endpoint | `localhost:4566` | APIs oficiais da AWS |
| Credenciais | `test`/`test` | Access Key + Secret Key reais |
| AMI | ID fixo no código | `data "aws_ami"` busca a mais recente |
| Região | `us-east-1` | `us-west-2` (ou qualquer outra) |
| Custo | Zero | Free Tier (t2.micro grátis por 12 meses) |

O bloco `data "aws_ami"` do tutorial não funciona no LocalStack Community porque ele não tem o catálogo de AMIs da AWS. Por isso hardcodei o ID `ami-0026a04369a3093cc`, que é exatamente o que o tutorial retorna. Na AWS real, o `data` block é a prática recomendada pra não ficar com ID defasado.

---

## Referências

- [HashiCorp — Create Infrastructure (tutorial seguido)](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-create)
- [Documentação do Terraform](https://developer.hashicorp.com/terraform/docs)
- [AWS Provider no Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [LocalStack no GitHub](https://github.com/localstack/localstack)
- [Docker — Get Started](https://docs.docker.com/get-started/)