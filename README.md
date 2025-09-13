#!/bin/bash
# Скрипт для развертывания n8n на Ubuntu VPS с поддержкой traefik и n8n-mcp
# Цвета для вывода
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Глобальные переменные
PROJECT_DIR="n8n-compose"
DOMAIN=""
MAIN_DOMAIN=""
SUBDOMAIN=""
SSL_EMAIL=""
ORIGINAL_DIR=""
PROJECT_FULL=""
N8N_BASIC_AUTH_ACTIVE="true"
N8N_BASIC_AUTH_USER=""
N8N_BASIC_AUTH_PASSWORD=""
N8N_ENCRYPTION_KEY=""
MCP_PORT="3000"
N8N_API_KEY=""
MCP_AUTH_TOKEN=""
LOG_LEVEL="info"
VERSION="latest"
GITHUB_REPOSITORY="czlonkowski/n8n-mcp"

# Функции для вывода сообщений
print_header() {
    echo -e "${BLUE}================================${NC}"
    echo -e "${BLUE}$1${NC}"
    echo -e "${BLUE}================================${NC}"
}

print_success() {
    echo -e "${GREEN}✓ $1${NC}"
}

print_warning() {
    echo -e "${YELLOW}⚠ $1${NC}"
}

print_error() {
    echo -e "${RED}✗ $1${NC}"
}

print_info() {
    echo -e "${BLUE}ℹ $1${NC}"
}

# Проверка версии Ubuntu
check_ubuntu_version() {
    print_header "Проверка версии Ubuntu"
    if [ -f /etc/os-release ]; then
        . /etc/os-release
        case $VERSION_ID in
            "20.04"|"22.04"|"24.04")
                print_success "Обнаружена поддерживаемая версия Ubuntu $VERSION_ID LTS"
                ;;
            *)
                print_error "Обнаружена неподдерживаемая версия Ubuntu: $VERSION_ID"
                echo "Поддерживаемые версии: Ubuntu 20.04, 22.04, 24.04 LTS"
                exit 1
                ;;
        esac
    else
        print_error "Не удалось определить версию Ubuntu"
        exit 1
    fi
}

# Проверка и установка Docker
install_docker() {
    print_header "Проверка и установка Docker"
    if command -v docker &> /dev/null; then
        DOCKER_VERSION=$(docker --version | cut -d' ' -f3 | cut -d',' -f1)
        print_success "Docker уже установлен: $DOCKER_VERSION"
        if docker compose version &> /dev/null || docker-compose version &> /dev/null; then
            if docker compose version &> /dev/null; then
                COMPOSE_VERSION=$(docker compose version | cut -d' ' -f4)
                print_success "Docker Compose v2 уже установлен: $COMPOSE_VERSION"
            else
                COMPOSE_VERSION=$(docker-compose version | cut -d' ' -f3)
                print_warning "Обнаружена Docker Compose v1: $COMPOSE_VERSION"
                print_info "Рекомендуется обновить до Docker Compose v2"
            fi
        else
            print_warning "Docker Compose не найден, устанавливаем..."
            install_docker_compose
        fi
    else
        print_warning "Docker не найден, устанавливаем..."
        install_docker_engine
    fi
}

# Установка Docker Engine
install_docker_engine() {
    print_info "Обновление пакетов..."
    apt update && apt upgrade -y
    print_info "Установка необходимых пакетов..."
    apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
    print_info "Добавление ключа Docker..."
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    print_info "Добавление репозитория Docker..."
    echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
    print_info "Установка Docker Engine..."
    apt update
    apt install -y docker-ce docker-ce-cli containerd.io
    print_info "Запуск Docker..."
    systemctl start docker
    systemctl enable docker
    print_success "Docker успешно установлен"
}

# Установка Docker Compose
install_docker_compose() {
    print_info "Установка Docker Compose v2..."
    apt install -y docker-compose-plugin
    print_success "Docker Compose успешно установлен"
}

# Проверка существующих контейнеров traefik, n8n и n8n-mcp
check_existing_containers() {
    print_header "Проверка существующих контейнеров traefik, n8n и n8n-mcp"
    # Проверка запущенных контейнеров
    if docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}" | grep -q "traefik\|n8n\|n8n-mcp"; then
        print_warning "Обнаружены существующие контейнеры:"
        docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}" | grep -E "traefik|n8n|n8n-mcp"
        read -p "Хотите остановить и удалить существующие контейнеры? (y/n): " -n 1 -r
        echo
        if [[ $REPLY =~ ^[Yy]$ ]]; then
            print_info "Останавливаем существующие контейнеры..."
            docker stop $(docker ps -aq --filter "name=traefik") 2>/dev/null || true
            docker stop $(docker ps -aq --filter "name=n8n") 2>/dev/null || true
            docker stop $(docker ps -aq --filter "name=n8n-mcp") 2>/dev/null || true
            docker rm $(docker ps -aq --filter "name=traefik") 2>/dev/null || true
            docker rm $(docker ps -aq --filter "name=n8n") 2>/dev/null || true
            docker rm $(docker ps -aq --filter "name=n8n-mcp") 2>/dev/null || true
            print_success "Существующие контейнеры остановлены и удалены"
        fi
    else
        print_success "Существующие контейнеры traefik/n8n/n8n-mcp не найдены"
    fi

    # Проверка Docker volumes
    if docker volume ls | grep -q "n8n_data\|traefik_data\|mcp_logs"; then
        print_warning "Обнаружены существующие volumes:"
        docker volume ls | grep -E "n8n_data|traefik_data|mcp_logs"
        read -p "Хотите сохранить существующие volumes? (y/n): " -n 1 -r
        echo
        if [[ ! $REPLY =~ ^[Yy]$ ]]; then
            print_info "Удаляем существующие volumes..."
            docker volume rm $(docker volume ls -q | grep -E "n8n_data|traefik_data|mcp_logs") 2>/dev/null || true
            print_success "Существующие volumes удалены"
        fi
    else
        print_success "Существующие volumes n8n_data/traefik_data/mcp_logs не найдены"
    fi

    # Проверка существующих SSL сертификатов
    if [ -d "$PROJECT_DIR" ] && [ -f "$PROJECT_DIR/letsencrypt/acme.json" ]; then
        print_warning "Обнаружены существующие SSL сертификаты в $PROJECT_DIR/letsencrypt/"
        read -p "Хотите сохранить существующие сертификаты? (y/n): " -n 1 -r
        echo
        if [[ ! $REPLY =~ ^[Yy]$ ]]; then
            print_info "Удаляем существующие сертификаты..."
            rm -rf "$PROJECT_DIR/letsencrypt"
            print_success "Существующие сертификаты удалены"
        else
            print_info "Существующие сертификаты будут сохранены"
        fi
    fi
}

# Настройка домена и SSL
setup_domain_ssl() {
    print_header "Настройка домена и SSL"
    read -p "Введите ваш домен (например: n8n.example.com): " DOMAIN
    if [ -z "$DOMAIN" ]; then
        print_error "Домен не может быть пустым"
        exit 1
    fi
    if [[ ! $DOMAIN =~ ^[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
        print_error "Неверный формат домена: $DOMAIN"
        exit 1
    fi
    read -p "Введите email для Let's Encrypt сертификата: " SSL_EMAIL
    if [ -z "$SSL_EMAIL" ]; then
        print_error "Email не может быть пустым"
        exit 1
    fi
    if [[ ! $SSL_EMAIL =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
        print_error "Неверный формат email: $SSL_EMAIL"
        exit 1
    fi
    print_success "Домен: $DOMAIN"
    print_success "Email: $SSL_EMAIL"
    if [[ $DOMAIN =~ ^([^.]+)\.(.*)$ ]]; then
        SUBDOMAIN="${BASH_REMATCH[1]}"
        MAIN_DOMAIN="${BASH_REMATCH[2]}"
    else
        SUBDOMAIN="n8n"
        MAIN_DOMAIN="$DOMAIN"
    fi
    print_info "Субдомен: $SUBDOMAIN"
    print_info "Основной домен: $MAIN_DOMAIN"
}

# Настройка секретов
setup_secrets() {
    print_header "Настройка секретов"
    read -p "Basic Auth User [admin]: " input_user
    N8N_BASIC_AUTH_USER=${input_user:-admin}
    read -s -p "Basic Auth Password: " N8N_BASIC_AUTH_PASSWORD
    echo
    if [ -z "$N8N_BASIC_AUTH_PASSWORD" ]; then
        print_error "Пароль обязателен"
        exit 1
    fi
    N8N_ENCRYPTION_KEY=$(openssl rand -base64 32)
    print_success "Encryption key сгенерирован"
    read -s -p "N8N API Key (оставьте пустым для генерации): " input_api_key
    echo
    if [ -z "$input_api_key" ]; then
        N8N_API_KEY=$(openssl rand -base64 32)
        print_success "API Key сгенерирован"
    else
        N8N_API_KEY="$input_api_key"
    fi
    read -s -p "MCP Auth Token (оставьте пустым для генерации): " input_token
    echo
    if [ -z "$input_token" ]; then
        MCP_AUTH_TOKEN=$(openssl rand -base64 32)
        print_success "Auth Token сгенерирован"
    else
        MCP_AUTH_TOKEN="$input_token"
    fi
    print_success "Секреты настроены"
}

# Создание конфигурационных файлов
create_config_files() {
    print_header "Создание конфигурационных файлов"
    if [ -d "$PROJECT_DIR" ]; then
        print_warning "Директория $PROJECT_DIR уже существует"
        read -p "Хотите перезаписать файлы? (y/n): " -n 1 -r
        echo
        if [[ ! $REPLY =~ ^[Yy]$ ]]; then
            print_info "Пропускаем создание файлов"
            return
        fi
    fi
    mkdir -p "$PROJECT_DIR"
    cd "$PROJECT_DIR"
    print_info "Создание .env файла..."
    cat > .env << EOF
# Конфигурация n8n
DOMAIN_NAME=$MAIN_DOMAIN
SUBDOMAIN=$SUBDOMAIN
SSL_EMAIL=$SSL_EMAIL
# Часовой пояс (можно изменить)
GENERIC_TIMEZONE=Europe/Moscow
# База данных (по умолчанию SQLite, для PostgreSQL раскомментируйте)
# DB_TYPE=postgresdb
# DB_POSTGRESDB_HOST=postgres
# DB_POSTGRESDB_PORT=5432
# DB_POSTGRESDB_DATABASE=n8n
# DB_POSTGRESDB_USER=n8n
# DB_POSTGRESDB_PASSWORD=secure_password_here

# Basic Auth
N8N_BASIC_AUTH_ACTIVE=$N8N_BASIC_AUTH_ACTIVE
N8N_BASIC_AUTH_USER=$N8N_BASIC_AUTH_USER
N8N_BASIC_AUTH_PASSWORD=$N8N_BASIC_AUTH_PASSWORD
N8N_ENCRYPTION_KEY=$N8N_ENCRYPTION_KEY

# n8n-mcp
MCP_PORT=$MCP_PORT
N8N_API_KEY=$N8N_API_KEY
MCP_AUTH_TOKEN=$MCP_AUTH_TOKEN
LOG_LEVEL=$LOG_LEVEL
VERSION=$VERSION
GITHUB_REPOSITORY=$GITHUB_REPOSITORY
EOF
    print_info "Создание docker-compose.yml..."
    cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  traefik:
    image: traefik
    restart: always
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
      - "--certificatesresolvers.mytlschallenge.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - n8n-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.rule=Host(traefik.${DOMAIN_NAME})"
      - "traefik.http.routers.traefik.entrypoints=websecure"
      - "traefik.http.routers.traefik.tls.certresolver=mytlschallenge"
      - "traefik.http.routers.traefik.service=api@internal"

  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    ports:
      - "127.0.0.1:5678:5678"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(${SUBDOMAIN}.${DOMAIN_NAME})"
      - "traefik.http.routers.n8n.tls=true"
      - "traefik.http.routers.n8n.entrypoints=web,websecure"
      - "traefik.http.routers.n8n.tls.certresolver=mytlschallenge"
      - "traefik.http.middlewares.n8n.headers.SSLRedirect=true"
      - "traefik.http.middlewares.n8n.headers.STSSeconds=315360000"
      - "traefik.http.middlewares.n8n.headers.browserXSSFilter=true"
      - "traefik.http.middlewares.n8n.headers.contentTypeNosniff=true"
      - "traefik.http.middlewares.n8n.headers.forceSTSHeader=true"
      - "traefik.http.middlewares.n8n.headers.SSLHost=${DOMAIN_NAME}"
      - "traefik.http.middlewares.n8n.headers.STSIncludeSubdomains=true"
      - "traefik.http.middlewares.n8n.headers.STSPreload=true"
      - "traefik.http.routers.n8n.middlewares=n8n@docker"
    environment:
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - N8N_RUNNERS_ENABLED=true
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - TZ=${GENERIC_TIMEZONE}
      - DB_TYPE=${DB_TYPE:-sqlite}
      - DB_POSTGRESDB_HOST=${DB_POSTGRESDB_HOST}
      - DB_POSTGRESDB_PORT=${DB_POSTGRESDB_PORT}
      - DB_POSTGRESDB_DATABASE=${DB_POSTGRESDB_DATABASE}
      - DB_POSTGRESDB_USER=${DB_POSTGRESDB_USER}
      - DB_POSTGRESDB_PASSWORD=${DB_POSTGRESDB_PASSWORD}
      - N8N_BASIC_AUTH_ACTIVE=${N8N_BASIC_AUTH_ACTIVE}
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/files
    networks:
      - n8n-network
    healthcheck:
      test: ["CMD", "sh", "-c", "wget --quiet --spider --tries=1 --timeout=10 http://localhost:5678/healthz || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  n8n-mcp:
    image: ghcr.io/${GITHUB_REPOSITORY}/n8n-mcp:latest
    container_name: n8n-mcp
    restart: unless-stopped
    ports:
      - "${MCP_PORT:-3000}:3000"
    environment:
      - NODE_ENV=production
      - N8N_MODE=true
      - MCP_MODE=http
      - N8N_API_URL=http://n8n:5678
      - N8N_API_KEY=${N8N_API_KEY}
      - MCP_AUTH_TOKEN=${MCP_AUTH_TOKEN}
      - AUTH_TOKEN=${MCP_AUTH_TOKEN}
      - LOG_LEVEL=${LOG_LEVEL}
    volumes:
      - ./data:/app/data:ro
      - mcp_logs:/app/logs
    networks:
      - n8n-network
    depends_on:
      n8n:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

volumes:
  n8n_data:
    driver: local
  traefik_data:
    driver: local
  mcp_logs:
    driver: local

networks:
  n8n-network:
    driver: bridge
EOF
    mkdir -p local-files data
    if [ -f "docker-compose.yml" ] && [ -f ".env" ]; then
        print_success "Конфигурационные файлы созданы в директории $PROJECT_DIR"
        if docker compose config --quiet 2>/dev/null; then
            print_success "Синтаксис docker-compose.yml корректен"
        else
            print_warning "Предупреждение: возможны проблемы с синтаксисом docker-compose.yml"
        fi
    else
        print_error "Ошибка: не удалось создать конфигурационные файлы"
        exit 1
    fi
    cd "$ORIGINAL_DIR" || {
        print_warning "Не удалось вернуться в исходную директорию: $ORIGINAL_DIR"
    }
}

# Проверка конфигурационных файлов
check_config_files() {
    print_info "Проверка конфигурационных файлов..."
    if [ ! -f "docker-compose.yml" ]; then
        print_error "Файл docker-compose.yml не найден"
        return 1
    fi
    if ! docker compose config --quiet 2>/dev/null; then
        print_error "Ошибка в синтаксисе docker-compose.yml"
        print_info "Проверьте файл docker-compose.yml на ошибки"
        return 1
    else
        print_success "docker-compose.yml корректен"
    fi
    if [ ! -f ".env" ]; then
        print_error "Файл .env не найден"
        return 1
    fi
    print_success "Все конфигурационные файлы найдены и корректны"
    return 0
}

# Запуск сервисов
start_services() {
    print_header "Запуск сервисов n8n, traefik и n8n-mcp"
    if [ ! -d "$PROJECT_DIR" ]; then
        print_error "Директория $PROJECT_DIR не найдена"
        exit 1
    fi
    cd "$PROJECT_DIR"
    if ! check_config_files; then
        exit 1
    fi
    print_info "Запуск Docker Compose..."
    docker compose up -d
    print_info "Ожидание запуска сервисов..."
    sleep 40
    if docker compose ps | grep -q "Up"; then
        print_success "Сервисы успешно запущены!"
        EXTERNAL_IP=$(curl -s ifconfig.me 2>/dev/null || echo "не удалось определить")
        print_header "Информация о развертывании"
        echo -e "${GREEN}✓ n8n доступен по адресу: https://${SUBDOMAIN}.${MAIN_DOMAIN}${NC}"
        echo -e "${GREEN}✓ Traefik dashboard: https://traefik.${MAIN_DOMAIN}${NC}"
        echo -e "${GREEN}✓ n8n-mcp доступен по порту: ${MCP_PORT}${NC}"
        echo -e "${BLUE}ℹ Внешний IP сервера: $EXTERNAL_IP${NC}"
        echo -e "${YELLOW}⚠ Убедитесь, что DNS записи указывают на этот IP${NC}"
        echo -e "${YELLOW}⚠ Первый запуск может занять несколько минут${NC}"
    else
        print_error "Ошибка запуска сервисов"
        print_info "Проверьте логи:"
        docker compose logs
        exit 1
    fi
}

# Настройка автозапуска
setup_autostart() {
    print_header "Настройка автозапуска"
    cat > /etc/systemd/system/n8n.service << EOF
[Unit]
Description=n8n Workflow Automation
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=$PROJECT_FULL
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF
    systemctl daemon-reload
    systemctl enable n8n
    print_success "Автозапуск настроен"
    print_info "Сервисы будут автоматически запускаться при перезагрузке сервера"
}

# Основная функция
main() {
    if [ "$EUID" -ne 0 ]; then
        print_error "Этот скрипт должен быть запущен с правами root (sudo)"
        exit 1
    fi
    print_header "Установка n8n на Ubuntu VPS"
    print_info "Скрипт поддерживает Ubuntu 20.04, 22.04, 24.04 LTS"
    echo
    ORIGINAL_DIR=$(pwd)
    PROJECT_FULL="$ORIGINAL_DIR/$PROJECT_DIR"
    check_ubuntu_version
    install_docker
    check_existing_containers
    setup_domain_ssl
    setup_secrets
    create_config_files
    start_services
    setup_autostart
    print_header "Установка завершена!"
    print_success "n8n, traefik и n8n-mcp успешно развернуты на вашем сервере"
    print_info "Не забудьте настроить DNS записи для домена"
    print_info "Используйте 'docker compose logs -f' в директории $PROJECT_DIR для просмотра логов"
}

# Обработка аргументов командной строки
case "${1:-}" in
    --help|-h)
        echo "Скрипт для развертывания n8n на Ubuntu VPS"
        echo ""
        echo "Использование: $0 [опции]"
        echo ""
        echo "Опции:"
        echo "  --help, -h          Показать эту справку"
        echo "  --version, -v       Показать версию скрипта"
        echo "  --update            Обновить n8n и n8n-mcp до последней версии"
        echo "  --stop              Остановить все сервисы"
        echo "  --restart           Перезапустить сервисы"
        echo "  --check             Проверить конфигурационные файлы"
        echo ""
        exit 0
        ;;
    --version|-v)
        echo "n8n Installation Script v1.1.0"
        echo "Поддерживает Ubuntu 20.04, 22.04, 24.04 LTS"
        exit 0
        ;;
    --update)
        print_header "Обновление n8n и n8n-mcp"
        ORIGINAL_DIR=$(pwd)
        if [ -d "n8n-compose" ]; then
            cd n8n-compose
            if [ ! -f "docker-compose.yml" ]; then
                print_error "Файл docker-compose.yml не найден в директории n8n-compose"
                exit 1
            fi
            docker compose pull
            docker compose up -d
            print_success "n8n и n8n-mcp обновлены"
        else
            print_error "Директория n8n-compose не найдена"
            exit 1
        fi
        cd "$ORIGINAL_DIR" 2>/dev/null || true
        exit 0
        ;;
    --stop)
        print_header "Остановка сервисов"
        ORIGINAL_DIR=$(pwd)
        if [ -d "n8n-compose" ]; then
            cd n8n-compose
            if [ ! -f "docker-compose.yml" ]; then
                print_error "Файл docker-compose.yml не найден в директории n8n-compose"
                exit 1
            fi
            docker compose down
            print_success "Сервисы остановлены"
        else
            print_error "Директория n8n-compose не найдена"
            exit 1
        fi
        cd "$ORIGINAL_DIR" 2>/dev/null || true
        exit 0
        ;;
    --restart)
        print_header "Перезапуск сервисов"
        ORIGINAL_DIR=$(pwd)
        if [ -d "n8n-compose" ]; then
            cd n8n-compose
            if [ ! -f "docker-compose.yml" ]; then
                print_error "Файл docker-compose.yml не найден в директории n8n-compose"
                exit 1
            fi
            docker compose restart
            print_success "Сервисы перезапущены"
        else
            print_error "Директория n8n-compose не найдена"
            exit 1
        fi
        cd "$ORIGINAL_DIR" 2>/dev/null || true
        exit 0
        ;;
    --check)
        print_header "Проверка конфигурации"
        ORIGINAL_DIR=$(pwd)
        if [ -d "n8n-compose" ]; then
            cd n8n-compose
            if check_config_files; then
                print_success "Конфигурация корректна"
            else
                print_error "Найдены ошибки в конфигурации"
                exit 1
            fi
        else
            print_error "Директория n8n-compose не найдена"
            exit 1
        fi
        cd "$ORIGINAL_DIR" 2>/dev/null || true
        exit 0
        ;;
    *)
        main
        ;;
esac
