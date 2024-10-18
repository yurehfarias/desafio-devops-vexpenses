# Provisionamento de Infraestrutura na AWS com Terraform

Este projeto utiliza o Terraform para criar uma infraestrutura básica na AWS, incluindo uma VPC, Subnet, Grupo de Segurança, Key Pair e uma instância EC2 com o servidor web Nginx instalado.

## Objetivo do Projeto

O objetivo deste projeto é demonstrar como provisionar e configurar uma infraestrutura na AWS utilizando Terraform de forma automatizada, seguindo boas práticas de segurança e organização.

## Pré-requisitos

- **Conta na AWS**: Você deve ter uma conta ativa na AWS com permissões adequadas para criar recursos como VPCs, instâncias EC2 e grupos de segurança.
- **Terraform**: Certifique-se de que o Terraform está instalado em sua máquina. Consulte a [documentação oficial](https://www.terraform.io/downloads.html) para obter instruções de instalação.
- **AWS CLI**: Instale a AWS CLI e configure suas credenciais. Isso pode ser feito seguindo as instruções na [documentação da AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html).

## Tecnologias Utilizadas

- **Terraform**: Ferramenta de infraestrutura como código para provisionamento de recursos na nuvem.
- **AWS**: Plataforma de computação em nuvem onde a infraestrutura será provisionada.
- **Nginx**: Servidor web que será instalado na instância EC2.

## Infraestrutura do Código

A infraestrutura do código Terraform é composta pelos seguintes componentes:

- **VPC (Virtual Private Cloud)**: Uma rede isolada onde os recursos serão criados.
- **Subnet**: Uma sub-rede dentro da VPC, que divide a rede em seções menores.
- **Grupo de Segurança**: Regras de firewall que controlam o tráfego de entrada e saída para a instância EC2.
- **Key Pair**: Par de chaves SSH para acesso seguro à instância EC2.
- **Instância EC2**: Máquina virtual na AWS onde o Nginx será instalado.

## Estrutura do Código

O código está contido em um arquivo chamado `main.tf` e é dividido nas seguintes seções:

```hcl
provider "aws" {
  region = "us-east-1" # Define a região da AWS onde os recursos serão criados
}

# Criada a Key Pair
resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"  # Algoritmo de criptografia RSA
  rsa_bits  = 2048   # Tamanho da chave RSA
}

resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "private-key"                      # Nome da chave que será usada na instância EC2
  public_key = tls_private_key.ec2_key.public_key_openssh # Chave pública gerada a partir da chave privada
}

# Criada a VPC
resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16" # Define o bloco CIDR para a VPC
  enable_dns_support   = true          # Habilita suporte a DNS
  enable_dns_hostnames = true          # Habilita nomes de host DNS

  tags = {
    Name = "main-vpc" # Nome da VPC
  }
}

# Criada a Subnet
resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id  # ID da VPC onde a subnet será criada
  cidr_block        = "10.0.1.0/24"         # Bloco CIDR para a subnet
  availability_zone = "us-east-1a"          # Zona de disponibilidade para a subnet

  tags = {
    Name = "main-subnet" # Nome da subnet
  }
}

# Criado o Security Group
resource "aws_security_group" "main_sg" {
  name   = "main-sg"           # Nome do grupo de segurança
  vpc_id = aws_vpc.main_vpc.id  # ID da VPC onde o grupo de segurança será criado

  # Regras de entrada (Ingress)
  ingress {
    from_port   = 22              # Permite acesso ao SSH na porta 22
    to_port     = 22              # Até a porta 22
    protocol    = "tcp"           # Protocolo TCP
    cidr_blocks = ["177.143.23.188/32"] # Restringi acesso ao meu IP
  }

  # Regras de saída (Egress)
  egress {
    from_port   = 0                # Permite todo tipo de saída
    to_port     = 0
    protocol    = "-1"             # Todos os protocolos
    cidr_blocks = ["0.0.0.0/0"]    # Permite acesso a qualquer IP
  }

  tags = {
    Name = "main-sg" # Nome do grupo de segurança
  }
}

# Declaração da Data Resource para obter a AMI Debian 12
data "aws_ami" "debian12" {
  most_recent = true

  owners = ["136693071363"] # Proprietário da AMI do Debian

  filter {
    name   = "name"
    values = ["debian-12-*"]
  }

  filter {
    name   = "architecture"
    values = ["x86_64"] # Arquitetura da AMI
  }
}

# Criação da instância EC2
resource "aws_instance" "web_server" {
  ami                    = data.aws_ami.debian12.id # AMI Debian 12 usando um bloco de dados
  instance_type          = "t2.micro"                # Tipo de instância
  subnet_id              = aws_subnet.main_subnet.id  # ID da subnet onde a instância será criada
  key_name               = aws_key_pair.ec2_key_pair.key_name # Nome da chave para acessar a instância
  vpc_security_group_ids = [aws_security_group.main_sg.id] # Associar o grupo de segurança à instância

  # Definindo o tamanho do armazenamento da máquina
  root_block_device {
    volume_size           = 20   # Tamanho do volume em GB
    volume_type           = "gp2" # Tipo de volume
    delete_on_termination = true  # Excluir o volume ao encerrar a instância
  }

  # Instalação do Nginx ao iniciar a instância
  user_data = <<-EOF
                #!/bin/bash
                apt-get update -y         
                apt-get upgrade -y        
                apt-get install -y nginx  
                systemctl start nginx     
                systemctl enable nginx    
                EOF

  tags = {
    Name = "webserver-ec2" # Nome da instância EC2
  }
}

# Private Key para acesso à instância
output "private_key" {
  description = "Chave privada para acessar a instância EC2 foi colocada no folder da main.tf"
  value       = tls_private_key.ec2_key.private_key_pem # Saída da chave privada
  sensitive   = true  # Marca a saída como sensível para não exibir em logs
}

# Endereço IP da instância
output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.web_server.public_ip # Saída do IP público da instância
}
```
# Passos para Execução

1. **Instalação do Terraform**: Baixe e instale o Terraform a partir do site oficial. Verifique a instalação executando `terraform -version` no terminal.
2. **Configuração da AWS CLI**: Instale a AWS CLI a partir do site oficial. Configure a AWS CLI com suas credenciais usando o comando:
   ```bash
   aws configure
   ```
3. **Clonagem do Repositório**: Clone este repositório em sua máquina local:
   ```bash
   git clone <url-do-repositório>
   cd <diretório-do-repositório>
   ```
4. **Inicialização do Terraform**: No diretório onde está o arquivo `main.tf`, execute:
   ```bash
   terraform init
   ```
5. **Planejamento da Infraestrutura**: Para visualizar o que será criado, execute:
   ```bash
   terraform plan
   ```
6. **Aplicação do Plano**: Para criar os recursos na AWS, execute:
   ```bash
   terraform apply
   ```
   Confirme a criação pressionando `yes`.


**Este projeto foi desenvolvido por Yure Farias, como parte do desafio DevOps para estágio na VExpenses.**
