# Bash-скрипт для миграции PHP+PostgreSQL сервиса в Docker Compose

Вот полный bash-скрипт, который анализирует систему, собирает информацию об установленных RPM-пакетах и создает Docker-окружение для PHP приложения с PostgreSQL.

```bash
#!/bin/bash

# Конфигурационные переменные
APP_NAME="php_postgres_app"
APP_DIR="/var/www/${APP_NAME}"
DOCKER_DIR="/opt/${APP_NAME}"
PHP_VERSION="8.2"
POSTGRES_VERSION="13"
DB_NAME="app_db"
DB_USER="app_user"
DB_PASSWORD="db_pass123"
DB_PORT="5432"

# Цвета для вывода
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Функция для проверки ошибок
check_error() {
  if [ $? -ne 0 ]; then
    echo -e "${RED}Ошибка: $1${NC}"
    exit 1
  fi
}

# Проверка прав root
if [ "$(id -u)" -ne 0 ]; then
  echo -e "${RED}Этот скрипт должен быть запущен с правами root${NC}"
  exit 1
fi

echo -e "${GREEN}=== Начало миграции PHP+PostgreSQL сервиса в Docker ===${NC}"

## 1. Анализ системы
echo -e "${YELLOW}1. Анализ текущей системы...${NC}"

# Сбор информации о пакетах
RPM_PACKAGES=$(rpm -qa)
PHP_MODULES=$(php -m 2>/dev/null)
PHP_VERSION_FULL=$(php -v 2>/dev/null | head -n 1 | cut -d " " -f 2 | cut -d "." -f 1-2)
POSTGRES_CLIENT_INSTALLED=$(echo "$RPM_PACKAGES" | grep -qE "php-pgsql|php-pdo_pgsql"; echo $?)

# Определение версии PHP
if [ -n "$PHP_VERSION_FULL" ]; then
  PHP_VERSION=$PHP_VERSION_FULL
  echo -e "Обнаружена PHP версии: ${PHP_VERSION}"
else
  echo -e "${YELLOW}PHP не установлен или не настроен, будет использована версия по умолчанию: ${PHP_VERSION}${NC}"
fi

## 2. Создание структуры каталогов
echo -e "${YELLOW}2. Создание структуры каталогов...${NC}"
mkdir -p "${DOCKER_DIR}/src" "${DOCKER_DIR}/config" "${DOCKER_DIR}/backup"
check_error "Не удалось создать каталоги"

# Копирование файлов приложения
if [ -d "$APP_DIR" ]; then
  echo -e "Копирование файлов приложения из ${APP_DIR}"
  cp -a "${APP_DIR}/." "${DOCKER_DIR}/src/"
  check_error "Не удалось скопировать файлы приложения"
else
  echo -e "${RED}Исходный каталог приложения ${APP_DIR} не найден!${NC}"
  exit 1
fi

## 3. Создание Dockerfile на основе установленных пакетов
echo -e "${YELLOW}3. Создание Dockerfile...${NC}"

# Определение необходимых пакетов
SYSTEM_PACKAGES=""
if echo "$RPM_PACKAGES" | grep -q "php-pgsql"; then
  SYSTEM_PACKAGES+="libpq-dev "
fi
if echo "$RPM_PACKAGES" | grep -q "php-gd"; then
  SYSTEM_PACKAGES+="libpng-dev libjpeg-dev "
fi
if echo "$RPM_PACKAGES" | grep -q "php-curl"; then
  SYSTEM_PACKAGES+="libcurl4-openssl-dev "
fi
if echo "$RPM_PACKAGES" | grep -q "php-zip"; then
  SYSTEM_PACKAGES+="zip libzip-dev "
fi
if echo "$RPM_PACKAGES" | grep -q "php-xml"; then
  SYSTEM_PACKAGES+="libxml2-dev "
fi

# Создание Dockerfile
cat > "${DOCKER_DIR}/Dockerfile" << EOF
# Автоматически сгенерированный Dockerfile для PHP+PostgreSQL
FROM php:${PHP_VERSION}-apache

# Установка системных зависимостей
RUN apt-get update && apt-get install -y \\
    ${SYSTEM_PACKAGES} \\
    postgresql-client \\
    && rm -rf /var/lib/apt/lists/*

# Установка PHP расширений
RUN docker-php-ext-install pdo pdo_pgsql pgsql

$(if echo "$PHP_MODULES" | grep -q "gd"; then
  echo "RUN docker-php-ext-configure gd --with-jpeg \\"
  echo "    && docker-php-ext-install gd"
fi)

$(if echo "$PHP_MODULES" | grep -q "imagick"; then
  echo "RUN pecl install imagick && docker-php-ext-enable imagick"
fi)

# Настройка Apache
RUN a2enmod rewrite

# Копирование конфигураций PHP
$(if [ -d /etc/php.d ]; then
  echo "COPY config/php/*.ini /usr/local/etc/php/conf.d/"
fi)

WORKDIR /var/www/html
COPY src/ .

RUN chown -R www-data:www-data /var/www/html
EOF

check_error "Не удалось создать Dockerfile"

## 4. Создание docker-compose.yml
echo -e "${YELLOW}4. Создание docker-compose.yml...${NC}"

cat > "${DOCKER_DIR}/docker-compose.yml" << EOF
version: '3.8'

services:
  app:
    build: .
    ports:
      - "80:80"
    volumes:
      - ./src:/var/www/html
    depends_on:
      - postgres
    environment:
      DB_HOST: postgres
      DB_PORT: ${DB_PORT}
      DB_NAME: ${DB_NAME}
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
    restart: unless-stopped

  postgres:
    image: postgres:${POSTGRES_VERSION}
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pg_data:/var/lib/postgresql/data
    ports:
      - "${DB_PORT}:5432"
    restart: unless-stopped

volumes:
  pg_data:
EOF

check_error "Не удалось создать docker-compose.yml"

## 5. Миграция базы данных
if [ $POSTGRES_CLIENT_INSTALLED -eq 0 ]; then
  echo -e "${YELLOW}5. Миграция базы данных PostgreSQL...${NC}"
  
  # Проверка установки pg_dump
  if ! command -v pg_dump &> /dev/null; then
    echo -e "${YELLOW}Установка postgresql-client для создания дампа БД...${NC}"
    yum install -y postgresql
    check_error "Не удалось установить postgresql-client"
  fi
  
  # Создание дампа БД
  echo -e "Создание дампа базы данных ${DB_NAME}"
  sudo -u postgres pg_dump -Fc ${DB_NAME} > "${DOCKER_DIR}/backup/db.dump"
  check_error "Не удалось создать дамп базы данных"
  
  # Восстановление БД в контейнере
  echo -e "${YELLOW}Запуск Docker Compose...${NC}"
  cd "${DOCKER_DIR}" && docker-compose up -d
  check_error "Не удалось запустить Docker Compose"
  
  echo -e "Ожидание запуска PostgreSQL..."
  sleep 20
  
  echo -e "Восстановление базы данных из дампа"
  docker exec -i "${APP_NAME}-postgres" pg_restore -U ${DB_USER} -d ${DB_NAME} < backup/db.dump
  check_error "Не удалось восстановить базу данных"
else
  echo -e "${YELLOW}5. Пакеты PostgreSQL не обнаружены, миграция БД пропущена${NC}"
  cd "${DOCKER_DIR}" && docker-compose up -d
  check_error "Не удалось запустить Docker Compose"
fi

## 6. Завершение
echo -e "${GREEN}=== Миграция завершена успешно! ===${NC}"
echo -e "Приложение доступно по адресу: http://localhost"
echo -e "Каталог с Docker-проектом: ${DOCKER_DIR}"
echo -e "Для управления используйте:"
echo -e "  cd ${DOCKER_DIR}"
echo -e "  docker-compose stop/start/restart"
```

## Инструкция по использованию

1. Сохраните скрипт как `migrate_php_postgres.sh`
2. Дайте права на выполнение:
   ```bash
   chmod +x migrate_php_postgres.sh
   ```
3. Запустите с правами root:
   ```bash
   sudo ./migrate_php_postgres.sh
   ```

## Особенности скрипта

1. **Автоматический анализ системы**:
   - Определение установленных PHP модулей
   - Проверка наличия PostgreSQL клиента
   - Анализ версии PHP

2. **Создание Dockerfile**:
   - Автоматическое определение необходимых зависимостей
   - Поддержка популярных PHP расширений (gd, imagick и др.)
   - Копирование конфигураций PHP

3. **Миграция базы данных**:
   - Автоматическое создание дампа PostgreSQL
   - Восстановление БД в контейнере
   - Проверка успешности операций

4. **Гибкая конфигурация**:
   - Возможность изменения параметров в начале скрипта
   - Проверка ошибок на каждом этапе
   - Цветной вывод статусов

Для дополнительной настройки можно изменить переменные в начале скрипта (версии PHP/PostgreSQL, учетные данные БД и т.д.).