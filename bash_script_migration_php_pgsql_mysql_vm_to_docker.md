Вот готовый bash-скрипт для миграции PHP-сервиса с поддержкой PostgreSQL и MySQL в Docker Compose, с анализом установленных RPM-пакетов:

```bash
#!/bin/bash
set -e

# Конфигурация
APP_NAME="php_app"
APP_DIR="/var/www/${APP_NAME}"
DOCKER_DIR="/opt/${APP_NAME}"
PHP_VERSION="8.2"
DB_TYPE="" # auto-detected
MYSQL_VERSION="8.0"
POSTGRES_VERSION="13"
HOST_PORT="80"

# Цвета
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# Функции
error_exit() {
  echo -e "${RED}Ошибка: $1${NC}" >&2
  exit 1
}

check_root() {
  if [ "$(id -u)" -ne 0 ]; then
    error_exit "Этот скрипт должен быть запущен с правами root"
  fi
}

detect_db_type() {
  if rpm -q php-mysqlnd &>/dev/null; then
    DB_TYPE="mysql"
  elif rpm -q php-pgsql &>/dev/null; then
    DB_TYPE="postgresql"
  else
    error_exit "Не обнаружены ни MySQL, ни PostgreSQL клиенты для PHP"
  fi
  echo -e "${GREEN}Обнаружена СУБД: ${DB_TYPE}${NC}"
}

create_dirs() {
  echo -e "${YELLOW}Создание структуры каталогов...${NC}"
  mkdir -p "${DOCKER_DIR}"/{src,config,backup}
  cp -a "${APP_DIR}/." "${DOCKER_DIR}/src/" || error_exit "Не удалось скопировать файлы приложения"
}

generate_dockerfile() {
  echo -e "${YELLOW}Генерация Dockerfile...${NC}"
  
  # Определение зависимостей
  local packages=""
  rpm -q php-gd &>/dev/null && packages+="libpng-dev libjpeg-dev "
  rpm -q php-curl &>/dev/null && packages+="libcurl4-openssl-dev "
  rpm -q php-zip &>/dev/null && packages+="zip libzip-dev "
  rpm -q php-xml &>/dev/null && packages+="libxml2-dev "
  
  # Специфичные для СУБД пакеты
  if [ "$DB_TYPE" = "mysql" ]; then
    packages+="default-mysql-client "
  elif [ "$DB_TYPE" = "postgresql" ]; then
    packages+="postgresql-client "
  fi

  cat > "${DOCKER_DIR}/Dockerfile" << EOF
FROM php:${PHP_VERSION}-apache

# Системные пакеты
RUN apt-get update && apt-get install -y \\
    ${packages} \\
    && rm -rf /var/lib/apt/lists/*

# PHP расширения
RUN docker-php-ext-install pdo ${DB_TYPE}
EOF

  # Дополнительные расширения
  if php -m | grep -q gd; then
    echo "RUN docker-php-ext-configure gd --with-jpeg && docker-php-ext-install gd" >> "${DOCKER_DIR}/Dockerfile"
  fi
  if php -m | grep -q pgsql; then
    echo "RUN docker-php-ext-install pdo_pgsql pgsql" >> "${DOCKER_DIR}/Dockerfile"
  fi
  if php -m | grep -q mysqli; then
    echo "RUN docker-php-ext-install mysqli" >> "${DOCKER_DIR}/Dockerfile"
  fi

  cat >> "${DOCKER_DIR}/Dockerfile" << EOF

# Настройка Apache
RUN a2enmod rewrite

WORKDIR /var/www/html
COPY src/ .
RUN chown -R www-data:www-data /var/www/html
EOF
}

generate_compose_file() {
  echo -e "${YELLOW}Генерация docker-compose.yml...${NC}"
  
  local db_service=""
  if [ "$DB_TYPE" = "mysql" ]; then
    db_service=$(cat << EOF
  db:
    image: mysql:${MYSQL_VERSION}
    environment:
      MYSQL_ROOT_PASSWORD: \${DB_ROOT_PASSWORD:-rootpass}
      MYSQL_DATABASE: \${DB_NAME:-${APP_NAME}_db}
      MYSQL_USER: \${DB_USER:-${APP_NAME}_user}
      MYSQL_PASSWORD: \${DB_PASSWORD:-password}
    volumes:
      - db_data:/var/lib/mysql
    ports:
      - "3306:3306"
EOF
    )
  elif [ "$DB_TYPE" = "postgresql" ]; then
    db_service=$(cat << EOF
  db:
    image: postgres:${POSTGRES_VERSION}
    environment:
      POSTGRES_DB: \${DB_NAME:-${APP_NAME}_db}
      POSTGRES_USER: \${DB_USER:-${APP_NAME}_user}
      POSTGRES_PASSWORD: \${DB_PASSWORD:-password}
    volumes:
      - db_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
EOF
    )
  fi

  cat > "${DOCKER_DIR}/docker-compose.yml" << EOF
version: '3.8'

services:
  app:
    build: .
    ports:
      - "${HOST_PORT}:80"
    volumes:
      - ./src:/var/www/html
    depends_on:
      - db
    environment:
      DB_HOST: db
      DB_TYPE: ${DB_TYPE}
EOF

  if [ "$DB_TYPE" = "mysql" ]; then
    echo "      DB_PORT: 3306" >> "${DOCKER_DIR}/docker-compose.yml"
  elif [ "$DB_TYPE" = "postgresql" ]; then
    echo "      DB_PORT: 5432" >> "${DOCKER_DIR}/docker-compose.yml"
  fi

  cat >> "${DOCKER_DIR}/docker-compose.yml" << EOF
      DB_NAME: \${DB_NAME:-${APP_NAME}_db}
      DB_USER: \${DB_USER:-${APP_NAME}_user}
      DB_PASSWORD: \${DB_PASSWORD:-password}

${db_service}

volumes:
  db_data:
EOF
}

migrate_database() {
  echo -e "${YELLOW}Миграция базы данных...${NC}"
  
  if [ "$DB_TYPE" = "mysql" ]; then
    if ! command -v mysqldump &>/dev/null; then
      echo -e "${YELLOW}Установка mysql-client...${NC}"
      yum install -y mysql || error_exit "Не удалось установить mysql-client"
    fi
    
    echo -e "Создание дампа MySQL..."
    mysqldump -u root -p "${APP_NAME}_db" > "${DOCKER_DIR}/backup/db.sql" || \
      error_exit "Не удалось создать дамп MySQL"
    
    echo -e "Восстановление в контейнере..."
    docker-compose -f "${DOCKER_DIR}/docker-compose.yml" up -d
    sleep 30 # Ожидаем запуск БД
    docker exec -i "${APP_NAME}-db" mysql -u root -prootpass "${APP_NAME}_db" < "${DOCKER_DIR}/backup/db.sql" || \
      error_exit "Не удалось восстановить дамп MySQL"
      
  elif [ "$DB_TYPE" = "postgresql" ]; then
    if ! command -v pg_dump &>/dev/null; then
      echo -e "${YELLOW}Установка postgresql-client...${NC}"
      yum install -y postgresql || error_exit "Не удалось установить postgresql-client"
    fi
    
    echo -e "Создание дампа PostgreSQL..."
    sudo -u postgres pg_dump "${APP_NAME}_db" > "${DOCKER_DIR}/backup/db.sql" || \
      error_exit "Не удалось создать дамп PostgreSQL"
    
    echo -e "Восстановление в контейнере..."
    docker-compose -f "${DOCKER_DIR}/docker-compose.yml" up -d
    sleep 20 # Ожидаем запуск БД
    docker exec -i "${APP_NAME}-db" psql -U "${APP_NAME}_user" -d "${APP_NAME}_db" < "${DOCKER_DIR}/backup/db.sql" || \
      error_exit "Не удалось восстановить дамп PostgreSQL"
  fi
}

main() {
  check_root
  echo -e "${GREEN}=== Начало миграции ${APP_NAME} ===${NC}"
  
  detect_db_type
  create_dirs
  generate_dockerfile
  generate_compose_file
  migrate_database
  
  echo -e "${GREEN}=== Миграция завершена успешно! ===${NC}"
  echo -e "Приложение доступно по адресу: http://localhost:${HOST_PORT}"
  echo -e "Каталог проекта: ${DOCKER_DIR}"
  echo -e "Для управления используйте:"
  echo -e "  cd ${DOCKER_DIR} && docker-compose [stop|start|restart]"
}

main
```

## Инструкция по использованию:

1. Сохраните скрипт как `migrate_php_to_docker.sh`
2. Дайте права на выполнение:
   ```bash
   chmod +x migrate_php_to_docker.sh
   ```
3. Запустите с правами root:
   ```bash
   sudo ./migrate_php_to_docker.sh
   ```

## Особенности скрипта:

1. **Автоматическое определение СУБД**:
   - Анализирует установленные RPM-пакеты для определения используемой БД (MySQL или PostgreSQL)
   - Поддерживает оба типа баз данных

2. **Генерация Dockerfile**:
   - Создает оптимальный Dockerfile на основе установленных PHP-модулей
   - Автоматически добавляет необходимые зависимости
   - Поддерживает популярные расширения (gd, pdo, mysqli и др.)

3. **Миграция данных**:
   - Создает дамп существующей базы данных
   - Восстанавливает данные в контейнере
   - Автоматически определяет нужные утилиты (mysqldump/pg_dump)

4. **Гибкая конфигурация**:
   - Все параметры можно изменить в начале скрипта
   - Подробный цветной вывод процесса
   - Проверка ошибок на каждом этапе

Для работы скрипта требуется установленный Docker и docker-compose. Скрипт автоматически установит необходимые клиенты для создания дампов БД, если они отсутствуют.