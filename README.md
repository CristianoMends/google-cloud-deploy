<div align="center"><img src="https://cloud.google.com/_static/cloud/images/social-icon-google-cloud-1200-630.png" alt="gcp logo" height="150px" /></div>

# Automação de Infraestrutura e CI/CD no Google Cloud com Terraform e GitHub Actions

Este tutorial explica como criar e automatizar a infraestrutura no Google Cloud usando Terraform. Além disso, implementamos um pipeline de CI/CD com GitHub Actions para implantar aplicações automaticamente em uma VM.

## Índice
- [Pré-requisitos](#pré-requisitos)
- [Passo 1: Configuração da Rede VPC](#passo-1-configuração-da-rede-vpc)
- [Passo 2: Criando uma Máquina Virtual (VM)](#passo-2-criando-uma-máquina-virtual-vm)
- [Passo 3: Configuração de Firewall](#passo-3-configuração-de-firewall)
- [Passo 4: Obtendo o IP Público da VM](#passo-4-obtendo-o-ip-público-da-vm)
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

## Passo 1: Configuração da Rede VPC
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

## Passo 2: Criando uma Máquina Virtual (VM)
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

## Passo 3: Configuração de Firewall
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

## Passo 4: Obtendo o IP Público da VM
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

