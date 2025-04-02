<div align="center">
  <img src="https://static-00.iconduck.com/assets.00/google-cloud-icon-2048x1646-7admxejz.png" alt="gcp logo" height="100px" />
  
  <img src="https://github.com/user-attachments/assets/49d78902-938b-47dd-901b-daeb1ec4efb0" alt="github actions logo" height="100px"/>
</div>

# Automação de Infraestrutura e CI/CD no Google Cloud com Terraform e GitHub Actions

Este tutorial explica como criar e automatizar a infraestrutura no Google Cloud usando Terraform. Além disso, implementamos um pipeline de CI/CD com GitHub Actions para implantar aplicações automaticamente em uma VM.

## Índice
- [Pré-requisitos](#pré-requisitos)
- [Passo 1: Organizando o Projeto Terraform](#passo-1-organizando-o-projeto-terraform)
- [Passo 2: Configuração da Rede VPC](#passo-2-configuração-da-rede-vpc)
- [Passo 3: Criando uma Máquina Virtual (VM)](#passo-3-criando-uma-máquina-virtual-vm)
- [Passo 4: Configuração de Firewall](#passo-4-configuração-de-firewall)
- [Passo 5: Obtendo o IP Público da VM](#passo-5-obtendo-o-ip-público-da-vm)
- [Como Aplicar o Terraform](#como-aplicar-o-terraform)
- [Conectando-se à VM](#conectando-se-à-vm)
- [Configurando CI/CD com GitHub Actions](#configurando-cicd-com-github-actions)
- [Criando um Workflow no GitHub Actions](#criando-um-workflow-no-github-actions)
- [Criando os Segredos no GitHub](#criando-os-segredos-no-github)
- [Como Funciona?](#como-funciona)

## Pré-requisitos
Antes de começar, certifique-se de ter os seguintes requisitos atendidos:
- Conta no Google Cloud Platform (GCP) com faturamento ativado.
- Google Cloud SDK instalado e autenticado.
- Terraform instalado.
- Acesso ao GitHub para configurar o CI/CD.

## Passo 1: Organizando o Projeto Terraform

Vamos começar criando uma estrutura de pastas para o seu projeto Terraform. Crie uma pasta no seu diretório local onde você armazenará todos os arquivos necessários para criar a infraestrutura.

### Estrutura do Projeto
```plaintext
meu-projeto-terraform/
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars
```
- main.tf: Arquivo principal que contém os recursos que vamos criar, como a rede, VM e regras de firewall.
- variables.tf: Onde declaramos variáveis, como o nome da VM, a região e a zona do GCP.
- outputs.tf: Contém as saídas que queremos exibir após a execução do Terraform, como o IP público da VM.
- terraform.tfvars: Arquivo onde você pode definir os valores das variáveis, como a chave SSH, a região, etc.

## Passo 2: Configuração da Rede VPC
A VPC (Virtual Private Cloud) permite a comunicação entre recursos na nuvem de forma isolada e segura.
```hcl
resource "google_compute_network" "vpc_network" {
  name                    = "my-vpc-network"
  auto_create_subnetworks = true
}
```
### Explicação:
- Criamos uma VPC chamada `my-vpc-network`.
- Permitimos a criação automática de sub-redes.

## Passo 3: Criando uma Máquina Virtual (VM)
A VM será criada na região `us-west1-a` com o tipo de máquina mais econômico (`f1-micro`).
```hcl
resource "google_compute_instance" "default" {
  name         = "simple-vm"
  machine_type = "f1-micro"
  zone         = "us-west1-a"

  tags = ["ssh"]

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  metadata_startup_script = <<-EOT
    sudo apt-get update
    sudo apt-get install -yq build-essential
  EOT

  network_interface {
    network = google_compute_network.vpc_network.id
    access_config {}  # Garante IP público
  }
}
```
### Explicação:
- Criamos uma VM chamada `simple-vm`.
- Atribuímos uma máquina econômica `f1-micro`.
- Definimos a zona `us-west1-a`.
- Usamos uma imagem Debian 11.
- Adicionamos um script de inicialização para instalar pacotes básicos.
- Associamos a VM à VPC criada e fornecemos um IP público.

## Passo 4: Configuração de Firewall
Para permitir o acesso SSH e a execução de uma aplicação Java, criamos duas regras de firewall.

### Regra para permitir SSH:
```hcl
resource "google_compute_firewall" "allow_ssh" {
  name    = "allow-ssh"
  network = google_compute_network.vpc_network.id

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["0.0.0.0/0"]
}
```

### Regra para permitir acesso à aplicação Java na porta 8080:
```hcl
resource "google_compute_firewall" "java" {
  name    = "java-app-firewall"
  network = google_compute_network.vpc_network.id

  allow {
    protocol = "tcp"
    ports    = ["8080"]
  }
  source_ranges = ["0.0.0.0/0"]
}
```

### Explicação:
- A regra `allow-ssh` permite conexões SSH de qualquer IP.
- A regra `java-app-firewall` permite tráfego TCP na porta 8080.

## Passo 5: Obtendo o IP Público da VM
Ao aplicar o Terraform, você verá o IP público da VM na saída.
```hcl
output "instance_ip" {
  value = google_compute_instance.default.network_interface.0.access_config.0.nat_ip
}
```

## Como Aplicar o Terraform
1. **Inicializar o Terraform** (executado apenas na primeira vez):
   ```sh
   terraform init
   ```
2. **Criar um plano de execução:**
   ```sh
   terraform plan
   ```
3. **Aplicar as configurações:**
   ```sh
   terraform apply -auto-approve
   ```

## Conectando-se à VM
Após a implantação, copie o IP público exibido e conecte-se via SSH:
```sh
ssh usuario@IP_DA_VM
```

## Configurando CI/CD com GitHub Actions
Agora, vamos configurar um **workflow no GitHub Actions** para automatizar o deploy na VM.

## Criando um Workflow no GitHub Actions
Crie um arquivo chamado `.github/workflows/deploy.yml` no seu repositório com o seguinte conteúdo:

```yaml
name: Deploy para VM no GCP

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Clonar o repositório
        uses: actions/checkout@v3

      - name: Conectar à VM e implantar
        uses: appleboy/ssh-action@v0.1.6
        with:
          host: ${{ secrets.VM_IP }}
          username: ${{ secrets.VM_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /caminho/do/projeto
            git pull origin main
            sudo systemctl restart minha-app
```

## Criando os Segredos no GitHub
Para que o workflow funcione, você precisa adicionar os seguintes **segredos** no GitHub:
1. **VM_IP** → O IP público da sua VM.
2. **VM_USER** → O usuário de SSH da sua VM (exemplo: `ubuntu` ou `gcp-user`).
3. **SSH_PRIVATE_KEY** → A chave privada SSH usada para conectar à VM.

## Como Funciona?
- Quando você faz **push** na branch `main`, o GitHub Actions conecta à VM.
- Ele faz um **git pull** para atualizar o código.
- Ele reinicia o serviço da aplicação automaticamente.

Agora sua aplicação será **implantada automaticamente** sempre que você fizer um push no GitHub! 🚀

