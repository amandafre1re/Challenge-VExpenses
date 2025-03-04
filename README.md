# Terraform AWS Infrastructure

## 1. Descrição Técnica do Arquivo `main.tf`
Este projeto define uma infraestrutura na **AWS** utilizando **Terraform**, criando os seguintes recursos:

### Recursos Criados
1. **VPC e Sub-rede**: Configuração de uma rede privada e uma sub-rede pública.
2. **Grupo de Segurança**: Permissões de acesso bem definidas para SSH e tráfego HTTP.
3. **Par de Chaves SSH**: Geração de chave privada e associação à instância.
4. **Tabela de Rotas e Internet Gateway**: Configuração para acesso à Internet.
5. **Instância EC2**: Servidor Debian 12 com **Nginx** instalado automaticamente.

---

## 2. Arquivo `main.tf` Modificado

```hcl
provider "aws" {
  region = "us-east-1"
}

variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}

variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "SeuNome"
}

resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}

resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}

resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}

resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}

resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH de IP específico e tráfego HTTP"
  vpc_id      = aws_vpc.main_vpc.id

  ingress {
    description = "Allow SSH from a specific IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["SEU_IP/32"]
  }

  ingress {
    description = "Allow HTTP traffic"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
}

resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]
  associate_public_ip_address = true

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get install -y nginx
              systemctl enable nginx
              systemctl start nginx
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}
```

---

## 3. Melhorias Implementadas

### **Aprimoramento da Segurança**
1. **Restrição do acesso SSH**: Em vez de permitir SSH de qualquer lugar (`0.0.0.0/0`), agora apenas **um IP específico** pode acessar.
2. **Habilitação apenas de serviços essenciais**: O acesso externo foi restrito apenas à porta **80 (HTTP)** para evitar exposição desnecessária.

### **Automação do Servidor Nginx**
1. **Instalação automática do Nginx** no momento da criação da instância.
2. **Configuração para iniciar automaticamente** após cada reinicialização do sistema.

###  **Outras Melhorias**
1. **Código mais organizado e legível**.
2. **Tags aprimoradas** para melhor rastreamento de recursos na AWS.

---

