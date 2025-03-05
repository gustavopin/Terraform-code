# Documentação - Desafio VExpanses - Estágio DevOPS

Este documento trás as tarefas a serem realizadas para o desafio proposto pela empresa VExpanses através do (documento provido)[https://drive.google.com/file/d/1otUEPPnify8i8nYS-h3vzSGJfk0u54IJ/view].

## Descrição do Arquivo Terraform Original

1. Provider

O comando "provider" conecta o progama à AWS. O argumento **region = "us-east-1"** declara qual data center da AWS será usado, neste caso o padrão, localizado na costa leste estadunidense.
```
provider "aws" {
  region = "us-east-1"
}
```

2. Variable

O comando *variable* declara as variáveis a serem usadas. Neste projeto seus nomes são "projeto" e "candidato".
```
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
```
Dentro do bloco das variáveis existem 3 argumentos:

        - *description*: é o propósito desta variável e o que é esperado nela;
        - *type*: é o tipo de variável (string, number, bool, map, entre outros). Para ambas as variáveis declaradas no código, está sendo usado o tipo *string*, um tipo de variável de texto;
        - *default*: utiliza este valor caso o usuário não digite nada ("VExpanses" para a variável "projeto" e "SeuNome" para a variável "candidato").

3. Resource

- Cada bloco de *resource* descreve uma (ou mais) infraestrutura. Como: compute instances, virtual networks, etc. O bloco de *resource* é composto, basicamente, de 4 argumentos fundamentais:

    - A declaração de *resource*;
    - O tipo de *resource* a ser usado;
    - O nome deste *resource*;
    - E entre as chaves {}, todos os argumentos a serem utilizados

```
resource "resource_type" "resource_name" {
    statement1 = "instance"
    statement2 = "bits"
}
```

No arquivo main.tf provido o primeiro resource está gerando uma chave de acesso criptografada de nome "ec2_key" usando o provedor TLS e o algoritmo RSA (um dos algoritmos de criptografia mais usados no mundo) e usa um tamanho de 2048 bits (descrito pela NITS como seguro até, pelo menos, 2030). Observando que: este bloco cria duas chaves, uma pública e uma privada, esta sendo de acesso exclusivo do usuário, enquanto a pública será usada a seguir.

 - A chave "ec2_key" criada no bloco *resource* anterior ainda não possui registro na AWS, o que é gerado no bloco:

```
resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}
```

Deste modo podemos criar a key pair dentro do EC2 da AWS chamado "ec2_key_pair". O argumento *key_name* faz com que este key pair tenha o nome referente às variáveis **projeto** e **candidato**. E o argumento *public_key* envia a chave para a AWS para que o EC2 reconheça a minha chave privada durante uma conexão.

- Com isso, podemos preparar a infraestrutura VPC (Virtual Private Cloud):

```
resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16" 
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}
```

Neste bloco vemos 5 partes principais:
        - A criação da VPC através do comando *aws_vpc*, que faz com que o Terraform use o provedor AWS para criar a VPC para o usuário de nome "aws_main";
        - A definição do block CIDR usando um block de 16 bits (aproximadamente 65000 variações de ips, desde 10.0.0.0 até 10.0.255.255);
        - Com o argumento *enable_dns_support* possibilitamos que o EC2 se comunique dentro da VPC usando nomes de domínios (Exemplo: servidorexemplo.ec2.internal) sem ele teríamos que usar diretamente os IPs gerados do bloco CIDR (Exemplo: 10.0.240.237). ALém disso, ;
        - *enable_dns_hostnames* faz com que o provedor AWS use o IP público (criado com o IP particular que será usado apenas internamente) e dê um nome "DNS" para ele: ec2.ippublico.compute-1.amazonaws.com, de modo que: ec2 define que estamos usando um Elastic Compute Cloud (servidor na nuvem), o *ippublico* é definido também durante a criação dos ips dentro da VPC juntamente com o IP privado, *compute-1* se refere à região do data center usado (no caso deste código *us-east-1*) e *amazonaws.com* é o domínio da AWS.
        - Por último, usamos nossas variáveis previamente estabelicidas como *tags* para nossa VPC, deixando ela mais organizada.

- Para a *subnet*:
O próximo passo do código é gerar uma subnet, um segmento da VPC para melhorar a gestão dentro de um único IP. No código main.tf a subnet é criada com o argumento (dentro de um *resource*) chamada *aws_subnet* e dado o nome de "main_subnet". Para identificar a VPC para a criação da subnet, usa-se o argumento *vpc_id = aws_vpc.main_vpc.id*, identificando a "vpc_main". É também definido (como padrão, mas pode ser modificado) o *cidr_block* como 10.0.1.0/24 (um IP com 24 bits) e a zona de disponibilidade (AZ / *availability  zone*) dentro da região do Data Center de escolha, ou seja, o último carctere "a" em "us-east-1a" define uma zona mais específica na região.

Por último, também são usadas as *tags* para a identificação e organização da subnet. Estas *tags* serão utilizadas à todo momento nos *resources* para organização.

- *Gateway* (Internet Gateway/IGW)
Este recurso possibilita a comunicação da VPC com a internet através do argumento *vpc_id = aws_vpc.main_vpc.id* (arguemnto que identifica a VPC dentro do main.tf). Este *gateway* é chamado de "main_igw" e é criado a partir do argumento *aws_internet_gateway*.

- *Route*
Porém, mesmo tendo uma *VPC*, *subnet* e um *gateway*, nosso sistema não consegue ter acesso à internet. Para isso temos o *resource* que cria uma *route* (comando dentro do nosso *resource*) de nome "main_route_table" a partir do argumento *aws_route_table*. O comando *route* também possui um argumento para seu id chamado *gateway_id* com o comando "aws_internet_gateway.main_igw.id", conectando-se ao nosso *gateway* e também possui um CIDR block com todas as possibilidades de IP (definido pelo 0.0.0.0/0) Deste modo, nosso VPC tem acesso à internet pública.

- *Table Association*
*aws_route_table_association* é um comando que liga nossa subnet, criada anteriormente, com a route table usando os argumentos de *subnet_id* e *route_table_id* (ambos identificando os recursos previamente criados). Deste modo, nossoa *subnet* sabe que rota usar para o tráfego de informações.

- Segurança para o EC2
Com o comando *aws_security_group* (nome *main_sg*) nosso código introduz um *Firewall* para o EC2, dando segurança às informações e decidindo o que/quem entra (regras definidos a partir do comando *ingress*) e o que/quem sai (regras definidas a partir do comando *egress*). Este comando acompanha um nome que usará os dados de nossas variáveis "projeto" e "candidato" juntamente com o sufixo "-sg" (*security group*) deste modo: VExpenses-SeuNome-sg, e uma conexão com nossa VPC pelo argumento *vpc_id*.

  - *Ingress*
  Como descrevi anteriormente, este comando é quem define o que e quem terá acesso ao nosso EC2, dado através do bloco de código:
  ```
  ingress {
    description      = "Allow SSH from anywhere"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
  ```
  Aqui definimos, manualmente, quem terá o acesso ao EC2 pelos argumentos *from_port* e *to_port*, que, no nosso caso, são declarados como a porta 22 (acesso restrito à administradores pelo Security Shell (SSH)) de qualquer lugar. O *protocol* define qual tipo de tráfego será usado para acesso (TCP é utilizado pela segurança). Por último são definidos os CIDR blocks de IPV4 ("0.0.0.0/0")  e IPV6 ("::/0").

  - *Egress*
  O bloco *egress* controla o tráfego do EC2. No arquivo main.tf temos:
  ```
    egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
  ```
  Neste código as regras de saída permitem que qualquer tráfego saia do EC2, os valores "0" para *from_port* e *to_port* permitem todos os tipos de tráfego (SSH, HTTP/HTTPS), juntamente com *protocol* sendo o valor "-1", permitindo todos os protocolos a serem usados (TCP, UDP, etc). Novamente, os CIDR blocks de IPV4 e IPV6 permitem qualquer IP nesta comunicação.

- AMI (Amazon Machine Image)
O bloco a seguir busca na AWS uma AMI em versão mais recente (*most_recent = true*) a partir de filtros (*filter{}*) estabelicidos:
```
data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}
```
O primeiro filtro procuro por AMIs que tenham o nome "debian-12-amd64-*", sendo que "*" ao final deste nome refere ao filtro: a AMI deve começar por "debian-12-amd64-" e a partir disso o último valor pode ser qualquer um. Exemplo: debian-12-amd64-20250305-1234. Nota-se que Debian 12 é uma distribuição do Linux, 12 sendo sua versão (Bookworm).

O segundo filtro define a virtualização como **HVM** (hardware virtualization), sendo mais atual e usando aceleração de hardware para melhor desempenho.

- Criação do EC2
Este bloco define a criação da instância EC2 assi mcomo as configurações de disco:
```
resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}
```

  - *ami = data.aws_ami.debian12.id*: define a AMI que será usada para a instância de EC2, a partir do data source debian12 criado anteriormente;
  - *instance_type = "t2.micro"*: estabelece o tipo de instância a ser utilizada, *t2.micro* nomeia a intância como sendo a "T2" do tipo "micro" (7 variações de nano à 2xlarge, micro é a segunda menor) que utiliza um processador escalável até 3.3GHz ((AWS)[https://aws.amazon.com/pt/ec2/instance-types/]);
  - *subnet_id = aws_subnet.main_subnet.id*: associa a EC2 à *subnet* (main_subnet) criada anteriormente na VPC;
  - *key_name = aws_key_pair.ec2_key_pair.key_name*: define as chaves para acesso ao EC2;
  - *security_groups = [aws_security_group.main_sg.name]*: engloba a instância EC2 ao *security group* criado anteriormente (*main_sg*);
  - *associate_public_ip_address = true*: garante um IP público à instância EC2.

Para o *root_block {}* (disco):
  - *volume_size = 20*: define o volume do disco (em gigabytes);
  - *volume_type = "gp2"*: estabelece o tipo do disco como "gp2" (SSD de uso geral);
  - *delete_on_termination = true*: permite que esse volume gerado seja deletado quando a instância for **terminada**.

Bloco *user_data*

Executa um script de atualização da instância Debian
  - *#!/bin/bash*: indica a inicilização do script no bash;
  - *apt-get update -y*: busca atualizações para os pacotes necessários;
  - *apt-get upgrade -y*: atualiza os pacotes.
  - O argumento "EOF" indica o começo e fim de um script dentro do terraform.

- *Outputs*
Os *outputs* são blocos para mostrar informações da infraestrutura:
```
output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}

output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}
```

No primeiro *output* (nomeado "private_key"):
  - *value = tls_private_key.ec2_key.private_key_pem*: define o valor do *output*, neste caso a chave privada criada anteriormente pelo recurso *private_key*;
  - *sensite = true*: marca este output como "sensível", ou seja, é uma informação que não pode vir à público e portanto não será exibida no terminal.

Para o segundo *output* ("ec2_public_ip"):
  - *value = aws_instance.debian_ec2.public_ip*: define o valor do endereço IP público da instância debian_ec2 criada.




#### Sugestão para seguranças
 - Criar um S3 bucket (storage do arquivo plan) com uma dynamodb table para não ter conflitos de atualizações durante uma mudança.
 - Criar uma passphrase como segunda camada de proteção durante a criação da key pair no primeiro block de resource (criação da keypair). ISSO PODE SER UM PROBLEMA SE O USUÁRIO ESQUECER A PASSPHRASE POIS O TERRAFORM NÃO GUARDA ELA NOS ARQUIVOS.
 - Dentro das regras de ingress, olhar os CIDR e IPV6 CIDR
 - Ver VPC endpoints para maior segurança no SG
 - Ver uso de porta 443 ou 80 no Egress