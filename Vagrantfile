Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.boot_timeout = 600

  # =========================
  # VM 1 - Banco de dados
  # =========================
  config.vm.define "database" do |db|
    db.vm.hostname = "smartlib-database"
    db.vm.network "private_network", ip: "192.168.56.11"

    db.vm.provider "virtualbox" do |vb|
      vb.name = "SmartLib-Database"
      vb.memory = 2048
      vb.cpus = 1
    end

    db.vm.provision "shell", inline: <<-SHELL
      set -e

      echo "======================================"
      echo " CONFIGURANDO VM DO BANCO"
      echo "======================================"

      export DEBIAN_FRONTEND=noninteractive

      apt-get update
      apt-get install -y docker.io

      systemctl enable --now docker

      until docker info >/dev/null 2>&1; do
        echo "Aguardando Docker..."
        sleep 2
      done

      # Se o banco já existir, apenas garante que está iniciado.
      if docker ps -a --format '{{.Names}}' | grep -q '^smartlib-db$'; then
        docker start smartlib-db >/dev/null 2>&1 || true
      else
        echo "Baixando PostgreSQL 17..."
        docker pull postgres:17

        echo "Criando banco..."
        docker run -d \
          --name smartlib-db \
          --restart unless-stopped \
          -p 5432:5432 \
          -e POSTGRES_DB=biblioteca \
          -e POSTGRES_USER=user \
          -e POSTGRES_PASSWORD=password \
          postgres:17
      fi

      echo "Aguardando PostgreSQL ficar pronto..."

      for i in $(seq 1 60); do
        if docker exec smartlib-db pg_isready -U user -d biblioteca >/dev/null 2>&1; then
          echo "PostgreSQL funcionando!"
          break
        fi

        if [ "$i" -eq 60 ]; then
          echo "PostgreSQL não iniciou a tempo."
          docker logs smartlib-db
          exit 1
        fi

        sleep 2
      done

      echo ""
      echo "======================================"
      echo " BANCO CONFIGURADO COM SUCESSO"
      echo "======================================"
      echo "IP:       192.168.56.11"
      echo "Porta:    5432"
      echo "Database: biblioteca"
      echo "Usuario:  user"
      echo "======================================"
    SHELL
  end

  # =========================
  # VM 2 - Aplicação
  # =========================
  config.vm.define "app" do |app|
    app.vm.hostname = "smartlib-app"
    app.vm.network "private_network", ip: "192.168.56.10"

    # Acesso opcional pelo computador hospedeiro
    app.vm.network "forwarded_port", guest: 80, host: 3000, auto_correct: true
    app.vm.network "forwarded_port", guest: 8080, host: 8080, auto_correct: true

    app.vm.provider "virtualbox" do |vb|
      vb.name = "SmartLib-App"
      vb.memory = 3072
      vb.cpus = 2
    end

    app.vm.provision "shell", inline: <<-SHELL
      set -e

      echo "======================================"
      echo " CONFIGURANDO VM DA APLICACAO"
      echo "======================================"

      export DEBIAN_FRONTEND=noninteractive

      # Cria swap para evitar falta de memória durante o build
      if [ ! -f /swapfile ]; then
        echo "Criando swap de 2GB..."
        fallocate -l 2G /swapfile
        chmod 600 /swapfile
        mkswap /swapfile
        swapon /swapfile
        echo '/swapfile none swap sw 0 0' >> /etc/fstab
      fi

      apt-get update
      apt-get install -y git curl ca-certificates docker.io

      systemctl enable --now docker

      # Aguarda o Docker ficar disponível
      until docker info >/dev/null 2>&1; do
        echo "Aguardando Docker..."
        sleep 2
      done

      # Cria a rede Docker que será usada pela API, frontend e Redis
      docker network inspect smartlib-net >/dev/null 2>&1 || \
        docker network create smartlib-net

      # Baixa/atualiza o projeto
      rm -rf /opt/SistemaGestaoBiblioteca
      git clone https://github.com/Caioxlw/SistemaGestaoBiblioteca.git /opt/SistemaGestaoBiblioteca

      cd /opt/SistemaGestaoBiblioteca

      echo "Construindo imagem da API..."
      docker build -t smartlib-api ./backend

      echo "Construindo imagem do frontend..."
      docker build -t smartlib-frontend ./frontend

      # Remove containers antigos, caso o provisionamento seja executado novamente
      docker rm -f smartlib-frontend smartlib-api smartlib-redis 2>/dev/null || true

      echo "Subindo Redis..."
      docker run -d \
        --name smartlib-redis \
        --restart unless-stopped \
        --network smartlib-net \
        redis:7-alpine

      # Aguarda Redis
      until docker exec smartlib-redis redis-cli ping 2>/dev/null | grep -q PONG; do
        echo "Aguardando Redis..."
        sleep 2
      done

      echo "Subindo API..."
      docker run -d \
        --name smartlib-api \
        --restart unless-stopped \
        --network smartlib-net \
        --network-alias api \
        -p 8080:8080 \
        -e ConnectionStrings__DefaultConnection="Host=192.168.56.11;Port=5432;Database=biblioteca;Username=user;Password=password" \
        -e Redis__ConnectionString="smartlib-redis:6379" \
        -e Jwt__Secret='SmartLibSuperSecretKey2026!@#$%^&*()MinimoSegura32Chars' \
        -e Jwt__Issuer="SmartLib" \
        -e Jwt__Audience="SmartLibUsers" \
        -e Jwt__ExpirationMinutes="480" \
        -e ASPNETCORE_ENVIRONMENT="Development" \
        smartlib-api

      echo "Aguardando API..."
      for i in $(seq 1 60); do
        if curl -fsS http://localhost:8080/health >/dev/null 2>&1; then
          echo "API funcionando!"
          break
        fi

        if [ "$i" -eq 60 ]; then
          echo "A API não respondeu a tempo."
          docker logs smartlib-api
          exit 1
        fi

        sleep 2
      done

      echo "Subindo frontend..."
      docker run -d \
        --name smartlib-frontend \
        --restart unless-stopped \
        --network smartlib-net \
        -p 80:80 \
        smartlib-frontend

      echo ""
      echo "======================================"
      echo " SMARTLIB CONFIGURADO COM SUCESSO"
      echo "======================================"
      echo "Frontend: http://192.168.56.10"
      echo "Frontend: http://localhost:3000"
      echo "Swagger:  http://localhost:8080/swagger"
      echo "Health:   http://localhost:8080/health"
      echo "======================================"
    SHELL
  end
end
