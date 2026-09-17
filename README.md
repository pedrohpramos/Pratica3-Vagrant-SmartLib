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

- VirtualBox
- Vagrant
- Internet

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