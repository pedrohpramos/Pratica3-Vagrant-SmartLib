# -*- mode: ruby -*-
# vi: set ft=ruby :

# =============================================================================
#  Sistema de Gestão de Biblioteca — Vagrantfile Multi-VM (2 Máquinas Virtuais)
#  Repositório: https://github.com/Caioxlw/SistemaGestaoBiblioteca
# -----------------------------------------------------------------------------
#  Stack REAL da aplicação (verificada no código-fonte):
#    - Backend : C# / .NET 10 (ASP.NET Core) + EF Core + Npgsql   -> porta 8080
#    - Cache   : Redis 7 (usado pelo RedisCacheService e HealthChecks)
#    - Banco   : PostgreSQL (migrations aplicadas automaticamente no Program.cs)
#    - Frontend: HTML/CSS/JS estático + Nginx com proxy reverso p/ /api
#
#  Arquitetura das VMs:
#    1. VM 'banco' (192.168.56.11)
#       - PostgreSQL escutando em 0.0.0.0, banco 'biblioteca', usuário 'user'
#       - Redis escutando em 0.0.0.0 (porta 6379)
#
#    2. VM 'app' (192.168.56.10)
#       - .NET 10 SDK, build + publish do backend, serviço systemd na porta 8080
#       - Nginx (porta 80) servindo o frontend e fazendo proxy de
#         /api, /swagger e /health para a API local
#       - Configurada apontando para o banco/redis da VM 'banco'
#
#  Acesso de outra máquina na rede:
#    A porta 80 da VM 'app' é publicada em 0.0.0.0:8080 da máquina física.
#    Basta acessar http://<IP-DA-MAQUINA-QUE-RODOU-VAGRANT-UP>:8080
# =============================================================================

require "socket"

# Descobre o IP de rede (LAN/wifi) da máquina física, para mostrar no banner final.
# Não abre conexão de verdade: só pergunta ao SO qual interface sairia para fora.
def ip_do_host
  Socket.ip_address_list
        .select { |a| a.ipv4? && !a.ipv4_loopback? && !a.ipv4_multicast? }
        .map(&:ip_address)
        .reject { |ip| ip.start_with?("192.168.56.", "169.254.") } # host-only e link-local
        .first || "SEU-IP-LOCAL"
rescue StandardError
  "SEU-IP-LOCAL"
end

REPO_URL = "https://github.com/Caioxlw/SistemaGestaoBiblioteca.git"
IP_BANCO = "192.168.56.11"
IP_APP   = "192.168.56.10"
DB_NAME  = "biblioteca"
DB_USER  = "user"
DB_PASS  = "password"

# =============================================================================
#  Shell inline — VM BANCO (PostgreSQL + Redis)
# =============================================================================
$script_banco = <<-SHELL
  set -euo pipefail
  # Mostra exatamente qual comando falhou, em vez de abortar em silêncio
  trap 'rc=$?; echo ""; echo ">>>> [BANCO] ERRO: comando falhou na linha $LINENO (codigo $rc)"; echo ""; exit $rc' ERR
  export DEBIAN_FRONTEND=noninteractive

  echo "===================================================================="
  echo " [BANCO] 0/4 - Aguardando cloud-init e locks do apt/dpkg liberarem..."
  echo "===================================================================="
  command -v cloud-init &>/dev/null && cloud-init status --wait || true
  command -v fuser &>/dev/null || apt-get install -y psmisc >/dev/null 2>&1 || true
  for i in $(seq 1 60); do
    if ! command -v fuser &>/dev/null; then break; fi
    if ! fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1 \
       && ! fuser /var/lib/dpkg/lock >/dev/null 2>&1 \
       && ! fuser /var/cache/debconf/templates.dat >/dev/null 2>&1 \
       && ! fuser /var/lib/apt/lists/lock >/dev/null 2>&1; then
      break
    fi
    echo "Lock do apt/dpkg em uso, aguardando... ($i/60)"
    sleep 3
  done
  systemctl stop unattended-upgrades 2>/dev/null || true
  systemctl disable unattended-upgrades 2>/dev/null || true

  # Executa um comando repetindo em caso de lock temporario do apt/dpkg
  apt_retry() {
    local tries=0
    until "$@"; do
      tries=$((tries+1))
      if [ "$tries" -ge 10 ]; then
        echo ">>>> Comando falhou apos $tries tentativas: $*"
        return 1
      fi
      echo "Comando de apt falhou (tentativa $tries/10), aguardando lock liberar..."
      sleep 5
    done
  }
  export NEEDRESTART_MODE=l
  mkdir -p /etc/needrestart/conf.d
  echo '$nrconf{restart} = "l";' > /etc/needrestart/conf.d/99-disable.conf 2>/dev/null || true
  echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections

  echo "===================================================================="
  echo " [BANCO] 1/4 - Instalando PostgreSQL e Redis..."
  echo "===================================================================="
  apt_retry apt-get update -y
  apt_retry apt-get install -y postgresql postgresql-contrib redis-server curl ca-certificates

  systemctl enable postgresql && systemctl start postgresql

  echo "===================================================================="
  echo " [BANCO] 2/4 - Criando usuário '#{DB_USER}' e banco '#{DB_NAME}'..."
  echo "===================================================================="
  # Usuário/senha idênticos aos esperados pela connection string da aplicação
  sudo -u postgres psql -tc "SELECT 1 FROM pg_roles WHERE rolname = '#{DB_USER}'" | grep -q 1 || \
    sudo -u postgres psql -c "CREATE ROLE \\"#{DB_USER}\\" WITH LOGIN PASSWORD '#{DB_PASS}' CREATEDB;"
  sudo -u postgres psql -c "ALTER ROLE \\"#{DB_USER}\\" WITH PASSWORD '#{DB_PASS}';"

  sudo -u postgres psql -tc "SELECT 1 FROM pg_database WHERE datname = '#{DB_NAME}'" | grep -q 1 || \
    sudo -u postgres psql -c "CREATE DATABASE #{DB_NAME} OWNER \\"#{DB_USER}\\";"

  sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE #{DB_NAME} TO \\"#{DB_USER}\\";"
  # O EF Core cria as tabelas via Migrate(): precisa poder criar objetos no schema public
  sudo -u postgres psql -d #{DB_NAME} -c "ALTER SCHEMA public OWNER TO \\"#{DB_USER}\\";"
  sudo -u postgres psql -d #{DB_NAME} -c "GRANT ALL ON SCHEMA public TO \\"#{DB_USER}\\";"

  echo "===================================================================="
  echo " [BANCO] 3/4 - Liberando PostgreSQL para a rede privada..."
  echo "===================================================================="
  PG_CONF=$(find /etc/postgresql/ -name postgresql.conf | head -n 1)
  PG_HBA=$(find /etc/postgresql/ -name pg_hba.conf | head -n 1)

  sed -i "s/^#\\?listen_addresses\\s*=.*/listen_addresses = '*'/" "$PG_CONF"

  if ! grep -q "192.168.56.0/24" "$PG_HBA"; then
    echo "host    all             all             192.168.56.0/24         md5" >> "$PG_HBA"
  fi

  systemctl restart postgresql

  echo "===================================================================="
  echo " [BANCO] 4/4 - Configurando Redis para aceitar conexões da rede..."
  echo "===================================================================="
  sed -i "s/^bind .*/bind 0.0.0.0/" /etc/redis/redis.conf
  sed -i "s/^protected-mode .*/protected-mode no/" /etc/redis/redis.conf
  systemctl enable redis-server
  systemctl restart redis-server

  # Validação
  sudo -u postgres psql -d #{DB_NAME} -c "SELECT 'PostgreSQL pronto!' AS status;"
  redis-cli ping

  echo "===================================================================="
  echo " [BANCO] VM de dados pronta em #{IP_BANCO} (PG:5432 / Redis:6379)"
  echo "===================================================================="
SHELL

# =============================================================================
#  Shell inline — VM APP (.NET 10 + Nginx)
# =============================================================================
$script_app = <<-SHELL
  set -euo pipefail
  # Mostra exatamente qual comando falhou, em vez de abortar em silêncio
  trap 'rc=$?; echo ""; echo ">>>> [APP] ERRO: comando falhou na linha $LINENO (codigo $rc)"; echo ""; exit $rc' ERR
  export DEBIAN_FRONTEND=noninteractive

  echo "===================================================================="
  echo " [APP] 0/7 - Aguardando cloud-init e locks do apt/dpkg liberarem..."
  echo "===================================================================="
  command -v cloud-init &>/dev/null && cloud-init status --wait || true
  command -v fuser &>/dev/null || apt-get install -y psmisc >/dev/null 2>&1 || true
  for i in $(seq 1 60); do
    if ! command -v fuser &>/dev/null; then break; fi
    if ! fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1 \
       && ! fuser /var/lib/dpkg/lock >/dev/null 2>&1 \
       && ! fuser /var/cache/debconf/templates.dat >/dev/null 2>&1 \
       && ! fuser /var/lib/apt/lists/lock >/dev/null 2>&1; then
      break
    fi
    echo "Lock do apt/dpkg em uso, aguardando... ($i/60)"
    sleep 3
  done
  systemctl stop unattended-upgrades 2>/dev/null || true
  systemctl disable unattended-upgrades 2>/dev/null || true

  # Executa um comando repetindo em caso de lock temporario do apt/dpkg
  apt_retry() {
    local tries=0
    until "$@"; do
      tries=$((tries+1))
      if [ "$tries" -ge 10 ]; then
        echo ">>>> Comando falhou apos $tries tentativas: $*"
        return 1
      fi
      echo "Comando de apt falhou (tentativa $tries/10), aguardando lock liberar..."
      sleep 5
    done
  }
  export NEEDRESTART_MODE=l
  mkdir -p /etc/needrestart/conf.d
  echo '$nrconf{restart} = "l";' > /etc/needrestart/conf.d/99-disable.conf 2>/dev/null || true
  echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections

  echo "===================================================================="
  echo " [APP] 1/7 - Instalando dependências base e Nginx..."
  echo "===================================================================="
  apt_retry apt-get update -y
  apt_retry apt-get install -y curl git ca-certificates nginx libicu70 libssl3 postgresql-client

  echo "===================================================================="
  echo " [APP] 2/7 - Instalando .NET SDK 10 (o projeto exige net10.0)..."
  echo "===================================================================="
  export DOTNET_CLI_TELEMETRY_OPTOUT=1
  export DOTNET_NOLOGO=1
  export DOTNET_ROOT=/usr/share/dotnet
  export PATH="$PATH:/usr/share/dotnet"

  if ! /usr/share/dotnet/dotnet --list-sdks 2>/dev/null | grep -q "^10\\."; then
    echo "Baixando dotnet-install.sh..."
    curl --retry 5 --retry-delay 3 -fsSL https://dot.net/v1/dotnet-install.sh -o /tmp/dotnet-install.sh
    chmod +x /tmp/dotnet-install.sh

    # Tenta o canal 10.0; se indisponível, cai para o STS/LTS mais recente
    /tmp/dotnet-install.sh --channel 10.0 --install-dir /usr/share/dotnet --no-path || \
    /tmp/dotnet-install.sh --channel LTS  --install-dir /usr/share/dotnet --no-path
  fi

  ln -sf /usr/share/dotnet/dotnet /usr/bin/dotnet

  # Falha cedo e com mensagem clara se o SDK 10 não estiver disponível
  if ! dotnet --list-sdks | grep -q "^10\\."; then
    echo ""
    echo ">>>> ERRO: o SDK .NET 10 nao foi instalado. SDKs encontrados:"
    dotnet --list-sdks || true
    echo ">>>> O projeto tem <TargetFramework>net10.0</TargetFramework> e nao compila sem ele."
    exit 1
  fi
  echo "SDK instalado: $(dotnet --version)"

  echo "===================================================================="
  echo " [APP] 3/7 - Aguardando a VM do banco (#{IP_BANCO}:5432)..."
  echo "===================================================================="
  for i in $(seq 1 60); do
    if bash -c "cat < /dev/null > /dev/tcp/#{IP_BANCO}/5432" 2>/dev/null; then
      echo "PostgreSQL alcançável!"
      break
    fi
    echo "Aguardando banco... ($i/60)"
    sleep 3
  done

  echo "===================================================================="
  echo " [APP] 4/7 - Clonando o repositório em /opt/biblioteca..."
  echo "===================================================================="
  rm -rf /opt/biblioteca
  git clone --depth 1 #{REPO_URL} /opt/biblioteca

  echo "===================================================================="
  echo " [APP] 5/7 - Compilando e publicando o backend (.NET)..."
  echo "        (a primeira execução baixa os pacotes NuGet e pode demorar)"
  echo "===================================================================="
  cd /opt/biblioteca/backend
  export HOME=/root
  export DOTNET_ROOT=/usr/share/dotnet
  dotnet restore SistemaGestaoBiblioteca.csproj --verbosity minimal
  dotnet publish SistemaGestaoBiblioteca.csproj -c Release -o /opt/biblioteca/publish --no-restore --verbosity minimal

  if [ ! -f /opt/biblioteca/publish/SistemaGestaoBiblioteca.dll ]; then
    echo ">>>> ERRO: o publish nao gerou SistemaGestaoBiblioteca.dll"
    ls -la /opt/biblioteca/publish || true
    exit 1
  fi

  # Variáveis de ambiente da API.
  # O ASP.NET Core lê "ConnectionStrings:DefaultConnection" como
  # ConnectionStrings__DefaultConnection, apontando para a VM do banco.
  cat << 'EOF' > /opt/biblioteca/api.env
ASPNETCORE_ENVIRONMENT=Development
ASPNETCORE_URLS=http://0.0.0.0:8080
DOTNET_CLI_TELEMETRY_OPTOUT=1
Jwt__Secret=SmartLibSuperSecretKey2026!@#$%^&*()MinimoSegura32Chars
Jwt__Issuer=SmartLib
Jwt__Audience=SmartLibUsers
Jwt__ExpirationMinutes=480
EOF
  echo "ConnectionStrings__DefaultConnection=Host=#{IP_BANCO};Port=5432;Database=#{DB_NAME};Username=#{DB_USER};Password=#{DB_PASS}" >> /opt/biblioteca/api.env
  echo "Redis__ConnectionString=#{IP_BANCO}:6379" >> /opt/biblioteca/api.env

  chown -R vagrant:vagrant /opt/biblioteca

  echo "===================================================================="
  echo " [APP] 6/7 - Criando serviço systemd da API..."
  echo "===================================================================="
  cat << 'EOF' > /etc/systemd/system/biblioteca-api.service
[Unit]
Description=Sistema Gestao Biblioteca - API (.NET)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=vagrant
WorkingDirectory=/opt/biblioteca/publish
EnvironmentFile=/opt/biblioteca/api.env
ExecStart=/usr/bin/dotnet /opt/biblioteca/publish/SistemaGestaoBiblioteca.dll
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

  systemctl daemon-reload
  systemctl enable biblioteca-api
  systemctl restart biblioteca-api

  echo "===================================================================="
  echo " [APP] 7/7 - Publicando o frontend e configurando o Nginx..."
  echo "===================================================================="
  rm -rf /var/www/biblioteca
  mkdir -p /var/www/biblioteca
  cp -r /opt/biblioteca/frontend/. /var/www/biblioteca/
  rm -f /var/www/biblioteca/Dockerfile /var/www/biblioteca/nginx.conf /var/www/biblioteca/.dockerignore
  chown -R www-data:www-data /var/www/biblioteca

  # Mesma lógica do nginx.conf do projeto, mas apontando para a API local
  cat << 'EOF' > /etc/nginx/sites-available/biblioteca
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    root /var/www/biblioteca;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8080/api/;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /swagger {
        proxy_pass http://127.0.0.1:8080/swagger;
        proxy_set_header Host       $host;
        proxy_set_header X-Real-IP  $remote_addr;
    }

    location /health {
        proxy_pass http://127.0.0.1:8080/health;
        proxy_set_header Host       $host;
        proxy_set_header X-Real-IP  $remote_addr;
    }
}
EOF

  rm -f /etc/nginx/sites-enabled/default
  ln -sf /etc/nginx/sites-available/biblioteca /etc/nginx/sites-enabled/biblioteca
  nginx -t
  systemctl enable nginx
  systemctl restart nginx

  # Aguarda a API aplicar as migrations e responder
  for i in $(seq 1 40); do
    if curl -fs http://127.0.0.1:8080/health > /dev/null 2>&1; then
      echo "API respondendo em /health!"
      break
    fi
    echo "Aguardando API subir... ($i/40)"
    sleep 3
  done
  systemctl --no-pager status biblioteca-api | head -n 12 || true

  # ---------------------------------------------------------------------------
  # Banner final com o status real de cada serviço
  # ---------------------------------------------------------------------------
  st_api=$(systemctl is-active biblioteca-api || true)
  st_nginx=$(systemctl is-active nginx || true)
  if bash -c "cat < /dev/null > /dev/tcp/#{IP_BANCO}/5432" 2>/dev/null; then st_pg="ok"; else st_pg="SEM RESPOSTA"; fi
  if bash -c "cat < /dev/null > /dev/tcp/#{IP_BANCO}/6379" 2>/dev/null; then st_redis="ok"; else st_redis="SEM RESPOSTA"; fi

  echo ""
  echo "######################################################################"
  echo "#                                                                    #"
  echo "#            AMBIENTE BIBLIOTECA PROVISIONADO COM SUCESSO            #"
  echo "#                                                                    #"
  echo "######################################################################"
  echo ""
  echo "  SERVIÇOS"
  echo "  ------------------------------------------------------------------"
  printf "    %-28s %s\\n" "API .NET (biblioteca-api)"  "$st_api"
  printf "    %-28s %s\\n" "Nginx (frontend + proxy)"   "$st_nginx"
  printf "    %-28s %s\\n" "PostgreSQL (#{IP_BANCO}:5432)" "$st_pg"
  printf "    %-28s %s\\n" "Redis (#{IP_BANCO}:6379)"      "$st_redis"
  echo ""
  echo "  PORTAS"
  echo "  ------------------------------------------------------------------"
  printf "    %-34s %s\\n" "VM app  : 80   (Nginx)"  "-> host 8080"
  printf "    %-34s %s\\n" "VM app  : 8080 (API .NET)" "-> host 5080"
  printf "    %-34s %s\\n" "VM banco: 5432 (PostgreSQL)" "rede privada"
  printf "    %-34s %s\\n" "VM banco: 6379 (Redis)"      "rede privada"
  echo ""
  echo "  ACESSO NA MÁQUINA HOST"
  echo "  ------------------------------------------------------------------"
  echo "    Frontend : http://localhost:8080"
  echo "    Swagger  : http://localhost:8080/swagger"
  echo "    Health   : http://localhost:8080/health"
  echo "    API      : http://localhost:8080/api/livros"
  echo ""
  echo "  IPs INTERNOS DAS VMs (rede privada 192.168.56.0/24)"
  echo "  ------------------------------------------------------------------"
  echo "    app   : #{IP_APP}      (http://#{IP_APP})"
  echo "    banco : #{IP_BANCO}"
  echo ""
  echo "######################################################################"
  echo ""
SHELL

Vagrant.configure("2") do |config|
  config.vm.box = ENV["VAGRANT_BOX"] || "ubuntu/jammy64"
  config.vm.boot_timeout = 600
  config.ssh.insert_key = false

  # ===========================================================================
  # 1. VM BANCO DE DADOS (PostgreSQL + Redis)
  # ===========================================================================
  config.vm.define "banco" do |banco|
    banco.vm.hostname = "biblioteca-db"
    banco.vm.network "private_network", ip: IP_BANCO
    banco.vm.synced_folder ".", "/vagrant", disabled: true

    banco.vm.provider "virtualbox" do |vb|
      vb.name   = "biblioteca-banco"
      vb.memory = 1024
      vb.cpus   = 1
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
    end

    banco.vm.provision "shell", inline: $script_banco
  end

  # ===========================================================================
  # 2. VM APLICAÇÃO (Backend .NET + Frontend via Nginx)
  # ===========================================================================
  config.vm.define "app" do |app|
    app.vm.hostname = "biblioteca-app"
    app.vm.network "private_network", ip: IP_APP

    # host_ip "0.0.0.0" -> outras máquinas da rede acessam pelo IP do host
    app.vm.network "forwarded_port", guest: 80,   host: 8080, host_ip: "0.0.0.0", auto_correct: true
    app.vm.network "forwarded_port", guest: 8080, host: 5080, host_ip: "0.0.0.0", auto_correct: true

    app.vm.synced_folder ".", "/vagrant", disabled: true

    app.vm.provider "virtualbox" do |vb|
      vb.name   = "biblioteca-app"
      vb.memory = 2560   # o build do .NET consome memória
      vb.cpus   = 2
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
    end

    app.vm.provision "shell", inline: $script_app

    # =========================================================================
    # Banner final impresso na MÁQUINA HOST, já com o IP de rede real
    # =========================================================================
    app.trigger.after :up do |trigger|
      trigger.name = "Acesso à aplicação"
      trigger.ruby do |_env, _machine|
        host_ip = ip_do_host
        puts ""
        puts "=" * 70
        puts "  BIBLIOTECA NO AR — tudo pronto, sem intervenção manual"
        puts "=" * 70
        puts ""
        puts "  NESTA MÁQUINA:"
        puts "    Frontend  ->  http://localhost:8080"
        puts "    Swagger   ->  http://localhost:8080/swagger"
        puts "    Health    ->  http://localhost:8080/health"
        puts ""
        puts "  DE OUTRO COMPUTADOR / CELULAR NA MESMA REDE:"
        puts "    Frontend  ->  http://IP_DA_SUA_MAQUINA:8080"
        puts "    Swagger   ->  http://IP_DA_SUA_MAQUINA:8080/swagger"
        puts ""
        puts "  PORTAS PUBLICADAS NO HOST:"
        puts "    8080  ->  Nginx (frontend + proxy /api) da VM app"
        puts "    5080  ->  API .NET direta da VM app"
        puts ""
        puts "  VMs:"
        puts "    app    #{IP_APP}   (backend .NET 8080 + nginx 80)"
        puts "    banco  #{IP_BANCO}   (PostgreSQL 5432 + Redis 6379)"
        puts ""
        puts "  Obs.: se as portas 8080/5080 já estiverem ocupadas, o Vagrant"
        puts "        escolhe outras automaticamente (auto_correct) e avisa acima."
        puts ""
        puts "  Comandos úteis:"
        puts "    vagrant ssh app -c 'systemctl status biblioteca-api'"
        puts "    vagrant ssh app -c 'journalctl -u biblioteca-api -f'"
        puts "    vagrant halt   |   vagrant destroy -f"
        puts "=" * 70
        puts ""
      end
    end
  end
end