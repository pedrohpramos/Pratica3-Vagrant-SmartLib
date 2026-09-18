# Prática 3 - Vagrant + SmartLib

Projeto da prática de administração de servidores.

## Objetivo

Subir automaticamente duas máquinas virtuais integradas:

- `app` (192.168.56.10): Frontend (Nginx) + Backend/API (.NET 10)
- `banco` (192.168.56.11): Banco de Dados (PostgreSQL) + Cache (Redis)

A aplicação utilizada como base é:

https://github.com/Caioxlw/SistemaGestaoBiblioteca

## Pré-requisitos

No computador hospedeiro:

- [Vagrant](https://www.vagrantup.com/) instalado
- Conexão com a internet
- Um provider de virtualização compatível (veja abaixo)

### Por sistema operacional

| SO | Provider | Box (automático) |
|---|---|---|
| Windows (x86_64) | VirtualBox | `ubuntu/jammy64` |
| macOS Intel | VirtualBox | `ubuntu/jammy64` |
| macOS Apple Silicon (M1/M2/M3/M4) | VMware Fusion, Parallels ou QEMU | `bento/ubuntu-22.04-arm64` |
| Linux (x86_64) | VirtualBox | `ubuntu/jammy64` |
| Linux (ARM64) | QEMU ou VMware | `bento/ubuntu-22.04-arm64` |

> **Nota para macOS/Linux com VirtualBox:** se o `vagrant up` falhar com erro de rede, execute:
> ```bash
> sudo mkdir -p /etc/vbox
> echo '* 192.168.56.0/24' | sudo tee /etc/vbox/networks.conf
> ```

> **Nota para Apple Silicon:** o VirtualBox não é compatível. Use VMware Fusion (gratuito para uso pessoal), Parallels ou QEMU:
> ```bash
> # Opção QEMU (open-source)
> brew install qemu
> vagrant plugin install vagrant-qemu
> vagrant up --provider=qemu
>
> # Opção VMware
> vagrant plugin install vagrant-vmware-desktop
> vagrant up --provider=vmware_desktop
> ```

## Executando

Dentro desta pasta:

```bash
vagrant up
```

O Vagrant irá provisionar duas VMs. O fluxo automático inclui:

**Na VM `banco`:**
1. Instalar PostgreSQL e Redis.
2. Criar o banco de dados `biblioteca` e o usuário da aplicação.
3. Liberar acessos para a rede privada.

**Na VM `app`:**
1. Instalar .NET 10 SDK, Nginx e dependências base.
2. Aguardar o banco de dados ficar disponível.
3. Clonar o projeto do GitHub.
4. Compilar e publicar a API (.NET).
5. Configurar e iniciar a API como um serviço em background (`systemd`).
6. Configurar o Nginx para servir o frontend estático e atuar como proxy reverso para a API local.
7. Deixar tudo disponível nas portas encaminhadas sem precisar acessar as VMs.

## Acessos

O ambiente é configurado para ser acessado diretamente pelo seu navegador no computador hospedeiro ou em qualquer dispositivo na mesma rede.

Frontend:
http://localhost:8080

Swagger (Documentação da API):
http://localhost:8080/swagger

Health Check:
http://localhost:8080/health

Se preferir acessar de outro dispositivo na mesma rede, use o IP da sua máquina hospedeira na porta 8080 (ex: `http://192.168.1.5:8080`).

## Arquitetura das VMs

Aplicação (`app`):
- IP: `192.168.56.10`
- Encaminhamento de Portas (Host -> Convidado):
  - `8080` -> `80` (Nginx: Frontend + Proxy para API)
  - `5080` -> `8080` (Acesso direto à API .NET)

Banco de Dados (`banco`):
- IP: `192.168.56.11`
- Serviços: PostgreSQL (5432) e Redis (6379) escutando na rede privada (192.168.56.0/24).

## Contas da aplicação

Admin:

- E-mail: `admin@smartlib.com`
- Senha: `Admin@123`

Bibliotecário:

- E-mail: `biblio@smartlib.com`
- Senha: `Biblio@123`

Aluno:

- E-mail: `aluno@smartlib.com`
- Senha: `Aluno@123`

## Comandos úteis

Subir o ambiente:

```bash
vagrant up
```

Ver status das VMs:

```bash
vagrant status
```

Desligar as VMs:

```bash
vagrant halt
```

Excluir as VMs (perde os dados do banco):

```bash
vagrant destroy -f
```

Executar novamente os scripts de configuração (sem recriar a VM do zero):

```bash
vagrant provision
```

Verificar os logs da API na VM da aplicação:
```bash
vagrant ssh app -c "journalctl -u biblioteca-api -f"
```

## Observação

O `vagrant ssh` não é necessário para utilizar a aplicação.

Ele deve ser usado apenas para manutenção, depuração ou para verificar os logs dos serviços dentro das VMs.