Вот усовершенствованный bash-скрипт для миграции PHP-сервиса с PostgreSQL/MySQL в Docker, который анализирует систему и автоматически генерирует оптимальную конфигурацию:

```bash
#!/bin/bash
set -eo pipefail

# Конфигурация (можно менять)
APP_NAME="my_php_app"
APP_DIR="/var/www/${APP_NAME}"
DOCKER_DIR="/opt/${APP_NAME}_docker"
PHP_VERSION="8.2"  # будет переопределено, если обнаружено
COMPOSE_VERSION="3.8"

# Цвета для вывода
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# Функции
die() {
  echo -e "${RED}Ошибка: $1${NC}" >&2
  exit 1
}

check_root() {
  [[ $EUID -eq 0 ]] || die "Этот скрипт требует прав root!"
}

detect_environment() {
  echo -e "${YELLOW}Анализ системы...${NC}"
  
  # Проверка PHP
  if command -v php &>/dev/null; then
    PHP_VERSION=$(php -v | head -n1 | awk '{print $2}' | cut -d. -f1-2)
    PHP_MODULES=$(php -m)
    PHP_INI_DIR=$(php --ini | grep "Scan for additional" | awk '{print $NF}')
    echo -e "${GREEN}Обнаружен PHP ${PHP_VERSION}${NC}"
  else
    die "PHP не установлен!"
  fi

  # Проверка СУБД
  DB_TYPE=""
  if rpm -q php-mysqlnd &>/dev/null || rpm -q mysql-server &>/dev/null; then
    DB_TYPE="mysql"
    MYSQL_VERSION=$(rpm -qi mysql-server | grep Version | awk '{print $3}' || echo "8.0")
  fi
  
  if rpm -q php-pgsql &>/dev/null || rpm -q postgresql-server &>/dev/null; then
    [[ -n "$DB_TYPE" ]] && DB_TYPE+="+"
    DB_TYPE+="postgresql"
    POSTGRES_VERSION=$(rpm -qi postgresql-server | grep Version | awk '{print $3}' || echo "13")
  fi

  [[ -z "$DB_TYPE" ]] && die "Не обнаружены ни MySQL, ни PostgreSQL!"
  echo -e "${GREEN}Обнаружены СУБД: ${DB_TYPE}${NC}"
}

prepare_directories() {
  echo -e "${YELLOW}Подготовка каталогов...${NC}"
  mkdir -p "${DOCKER_DIR}"/{src,config,backup}
  
  # Копирование приложения с сохранением прав
  if [[ -d "$APP_DIR" ]]; then
    rsync -a "${APP_DIR}/" "${DOCKER_DIR}/src/"
    # Копирование конфигов PHP
    [[ -d "$PHP_INI_DIR" ]] && cp "${PHP_INI_DIR}"/*.ini "${DOCKER_DIR}/config/"
  else
    die "Каталог приложения ${APP_DIR} не найден!"
  fi
}

generate_dockerfile() {
  echo -e "${YELLOW}Генерация Dockerfile...${NC}"
  
  # Определение необходимых пакетов
  local packages=""
  rpm -q php-gd &>/dev/null && packages+="libpng-dev libjpeg-dev "
  rpm -q php-curl &>/dev/null && packages+="libcurl4-openssl-dev "
  rpm -q php-zip &>/dev/null && packages+="zip libzip-dev "
  rpm -q php-xml &>/dev/null && packages+="libxml2-dev "
  rpm -q php-mbstring &>/dev/null && packages+="libonig-dev "
  
  # Web сервер
  local webserver="apache"
  if rpm -q nginx &>/dev/null; then
    webserver="fpm"
    packages+="nginx "
  fi

  cat > "${DOCKER_DIR}/Dockerfile" << EOF
# Автосгенерированный Dockerfile для ${APP_NAME}
FROM php:${PHP_VERSION}-${webserver}

# Установка системных пакетов
RUN apt-get update && apt-get install -y \\
    ${packages} \\
    && rm -rf /var/lib/apt/lists/*

# Установка расширений PHP
EOF

  # MySQL расширения
  if [[ "$DB_TYPE" == *"mysql"* ]]; then
    echo "RUN docker-php-ext-install pdo_mysql mysqli" >> "${DOCKER_DIR}/Dockerfile"
  fi

  # PostgreSQL расширения
  if [[ "$DB_TYPE" == *"postgresql"* ]]; then
    echo "RUN docker-php-ext-install pdo_pgsql pgsql" >> "${DOCKER_DIR}/Dockerfile"
  fi

  # Дополнительные расширения
  [[ "$PHP_MODULES" == *"gd"* ]] && \
    echo "RUN docker-php-ext-configure gd --with-jpeg && docker-php-ext-install gd" >> "${DOCKER_DIR}/Dockerfile"
  
  [[ "$PHP_MODULES" == *"imagick"* ]] && \
    echo "RUN pecl install imagick && docker-php-ext-enable imagick" >> "${DOCKER_DIR}/Dockerfile"

  # Копирование конфигов
  if ls "${DOCKER_DIR}/config"/*.ini &>/dev/null; then
    echo -e "\n# PHP конфигурация" >> "${DOCKER_DIR}/Dockerfile"
    echo "COPY config/*.ini /usr/local/etc/php/conf.d/" >> "${DOCKER_DIR}/Dockerfile"
  fi

  # Завершение Dockerfile
  cat >> "${DOCKER_DIR}/Dockerfile" << EOF

# Настройка рабочего каталога
WORKDIR /var/www/html
COPY src/ .

# Права
RUN chown -R www-data:www-data /var/www/html
EOF

  # Для Apache
  if [[ "$webserver" == "apache" ]]; then
    echo "RUN a2enmod rewrite" >> "${DOCKER_DIR}/Dockerfile"
  fi
}

generate_compose() {
  echo -e "${YELLOW}Создание docker-compose.yml...${NC}"
  
  local db_services="" volumes=""
  
  # MySQL сервис
  if [[ "$DB_TYPE" == *"mysql"* ]]; then
    db_services+=$(cat << EOF

  mysql:
    image: mysql:${MYSQL_VERSION}
    environment:
      MYSQL_ROOT_PASSWORD: \${DB_ROOT_PASSWORD:-rootpass}
      MYSQL_DATABASE: \${DB_NAME:-${APP_NAME}_db}
      MYSQL_USER: \${DB_USER:-${APP_NAME}_user}
      MYSQL_PASSWORD: \${DB_PASSWORD:-password}
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 5s
      retries: 5
EOF
    )
    volumes+="  mysql_data:\n"
  fi

  # PostgreSQL сервис
  if [[ "$DB_TYPE" == *"postgresql"* ]]; then
    db_services+=$(cat << EOF

  postgres:
    image: postgres:${POSTGRES_VERSION}
    environment:
      POSTGRES_DB: \${DB_NAME:-${APP_NAME}_db}
      POSTGRES_USER: \${DB_USER:-${APP_NAME}_user}
      POSTGRES_PASSWORD: \${DB_PASSWORD:-password}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U \${DB_USER:-${APP_NAME}_user}"]
      interval: 5s
      timeout: 5s
      retries: 5
EOF
    )
    volumes+="  postgres_data:\n"
  fi

  # Web сервер
  local webserver_config=""
  if rpm -q nginx &>/dev/null; then
    webserver_config=$(cat << EOF

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./src:/var/www/html
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - php
EOF
    )
  fi

  cat > "${DOCKER_DIR}/docker-compose.yml" << EOF
version: '${COMPOSE_VERSION}'

services:
  app:
    build: .
    ${webserver_config:+depends_on:${webserver_config}}
EOF

  if [[ -z "$webserver_config" ]]; then
    cat >> "${DOCKER_DIR}/docker-compose.yml" << EOF
    ports:
      - "80:80"
    volumes:
      - ./src:/var/www/html
EOF
  fi

  # Добавляем переменные окружения
  cat >> "${DOCKER_DIR}/docker-compose.yml" << EOF
    environment:
      DB_HOST: ${DB_TYPE%%+*}
      DB_PORT: ${DB_TYPE == *"mysql"* ? "3306" : "5432"}
      DB_NAME: \${DB_NAME:-${APP_NAME}_db}
      DB_USER: \${DB_USER:-${APP_NAME}_user}
      DB_PASSWORD: \${DB_PASSWORD:-password}
${db_services}
${webserver_config}

volumes:
${volumes}
EOF
}

migrate_database() {
  echo -e "${YELLOW}Миграция базы данных...${NC}"
  
  # MySQL миграция
  if [[ "$DB_TYPE" == *"mysql"* ]]; then
    if ! command -v mysqldump &>/dev/null; then
      echo -e "${YELLOW}Установка mysql-client...${NC}"
      yum install -y mysql || die "Не удалось установить mysql-client"
    fi
    
    echo -e "${GREEN}Создание дампа MySQL...${NC}"
    mysqldump --single-transaction -u root -p "${APP_NAME}_db" > "${DOCKER_DIR}/backup/mysql_dump.sql" || \
      die "Не удалось создать дамп MySQL"
  fi

  # PostgreSQL миграция
  if [[ "$DB_TYPE" == *"postgresql"* ]]; then
    if ! command -v pg_dump &>/dev/null; then
      echo -e "${YELLOW}Установка postgresql-client...${NC}"
      yum install -y postgresql || die "Не удалось установить postgresql-client"
    fi
    
    echo -e "${GREEN}Создание дампа PostgreSQL...${NC}"
    sudo -u postgres pg_dump -Fc "${APP_NAME}_db" > "${DOCKER_DIR}/backup/postgres_dump.dump" || \
      die "Не удалось создать дамп PostgreSQL"
  fi
}

start_services() {
  echo -e "${YELLOW}Запуск Docker-сервисов...${NC}"
  cd "$DOCKER_DIR" && docker-compose up -d || die "Ошибка при запуске docker-compose"

  # Импорт данных
  if [[ "$DB_TYPE" == *"mysql"* ]] && [[ -f "${DOCKER_DIR}/backup/mysql_dump.sql" ]]; then
    echo -e "${GREEN}Импорт MySQL данных...${NC}"
    sleep 30  # Ожидаем инициализацию БД
    docker-compose exec -T mysql mysql -u root -prootpass "${APP_NAME}_db" < "${DOCKER_DIR}/backup/mysql_dump.sql"
  fi

  if [[ "$DB_TYPE" == *"postgresql"* ]] && [[ -f "${DOCKER_DIR}/backup/postgres_dump.dump" ]]; then
    echo -e "${GREEN}Импорт PostgreSQL данных...${NC}"
    sleep 20
    docker-compose exec -T postgres pg_restore -U "${APP_NAME}_user" -d "${APP_NAME}_db" < "${DOCKER_DIR}/backup/postgres_dump.dump"
  fi
}

show_summary() {
  echo -e "${GREEN}\n=== Миграция завершена успешно! ===${NC}"
  echo -e "Приложение доступно по адресу: http://localhost"
  echo -e "Каталог проекта: ${DOCKER_DIR}"
  echo -e "Для управления используйте:"
  echo -e "  cd ${DOCKER_DIR} && docker-compose [stop|start|restart|logs]"
}

main() {
  check_root
  detect_environment
  prepare_directories
  generate_dockerfile
  generate_compose
  migrate_database
  start_services
  show_summary
}

main
```

## Ключевые особенности скрипта:

1. **Автоматический анализ системы**:
   - Определяет версию PHP и установленные модули
   - Обнаруживает используемую СУБД (MySQL, PostgreSQL или обе)
   - Анализирует конфигурационные файлы PHP

2. **Интеллектуальная генерация Dockerfile**:
   - Автоматически определяет необходимые зависимости
   - Поддерживает оба типа СУБД (MySQL и PostgreSQL)
   - Копирует существующие PHP-конфиги
   - Определяет веб-сервер (Apache или Nginx)

3. **Миграция данных**:
   - Создает дампы баз данных
   - Автоматически восстанавливает данные в контейнерах
   - Добавляет healthcheck для СУБД

4. **Гибкость и надежность**:
   - Подробное логирование с цветовой разметкой
   - Проверка ошибок на каждом этапе
   - Поддержка переменных окружения
   - Автоматическая установка недостающих утилит

5. **Безопасность**:
   - Проверка прав root
   - Использование volumes для хранения данных БД
   - Сохранение правильных прав доступа к файлам

Для использования скрипта:
1. Сохраните его как `migrate_php_to_docker.sh`
2. Дайте права на выполнение: `chmod +x migrate_php_to_docker.sh`
3. Запустите с правами root: `sudo ./migrate_php_to_docker.sh`

Скрипт автоматически создаст всю необходимую конфигурацию Docker и перенесет ваше PHP-приложение с базой данных.