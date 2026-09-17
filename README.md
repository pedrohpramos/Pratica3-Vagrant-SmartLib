# Prática 3 - Vagrant + SmartLib

Projeto da prática de administração de servidores.

## Objetivo

Subir automaticamente duas máquinas virtuais:

- `app`: frontend + backend/API + Redis
- `database`: PostgreSQL

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

O Vagrant irá:

1. Criar a VM do banco.
2. Instalar Docker.
3. Subir PostgreSQL.
4. Criar o banco `biblioteca`.
5. Criar a VM da aplicação.
6. Instalar Docker.
7. Baixar o projeto do GitHub.
8. Construir a imagem da API.
9. Construir a imagem do frontend.
10. Subir Redis.
11. Configurar a API para acessar o PostgreSQL na outra VM.
12. Subir o frontend com Nginx.
13. Deixar tudo disponível sem precisar entrar via `vagrant ssh`.

## Acessos

Frontend:

http://localhost:3000

ou:

http://192.168.56.10

Swagger:

http://localhost:8080/swagger

Health Check:

http://localhost:8080/health

## VMs

Aplicação:

- IP: `192.168.56.10`

Banco:

- IP: `192.168.56.11`

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

Subir:

```bash
vagrant up
```

Ver status:

```bash
vagrant status
```

Desligar:

```bash
vagrant halt
```

Excluir as VMs:

```bash
vagrant destroy -f
```

Executar novamente os scripts de configuração:

```bash
vagrant provision
```

## Observação

O `vagrant ssh` não é necessário para utilizar a aplicação.

Ele pode ser usado apenas para manutenção ou para verificar os serviços dentro das VMs.