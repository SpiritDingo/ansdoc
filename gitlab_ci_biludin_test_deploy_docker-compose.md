# Настройка GitLab CI/CD для PHP с Docker Compose

Вот полная конфигурация для сборки, тестирования и деплоя PHP-приложения с использованием Docker Compose в GitLab CI.

## `.gitlab-ci.yml` для PHP + Docker Compose

```yaml
stages:
  - prepare
  - lint
  - build
  - test
  - deploy

variables:
  DOCKER_HOST: tcp://docker:2375
  DOCKER_DRIVER: overlay2
  COMPOSER_ALLOW_SUPERUSER: 1
  COMPOSER_CACHE_DIR: /tmp/composer-cache
  APP_NAME: my-php-app
  PRODUCTION_HOST: example.com
  STAGING_HOST: staging.example.com

services:
  - docker:dind

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - vendor/
    - $COMPOSER_CACHE_DIR

## Этапы CI/CD

# Подготовка окружения
prepare:
  stage: prepare
  image: docker:latest
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker-compose -f docker-compose.ci.yml pull || true
  only:
    - merge_requests
    - main

# Линтинг кода
lint:
  stage: lint
  image: composer:latest
  script:
    - composer install --prefer-dist --no-ansi --no-interaction --no-progress
    - vendor/bin/phpcs --standard=PSR12 src/
    - vendor/bin/phpstan analyse --level=8 src/
  only:
    - merge_requests

# Сборка образов
build:
  stage: build
  image: docker:latest
  script:
    - docker-compose -f docker-compose.ci.yml build
    - docker-compose -f docker-compose.ci.yml push
  artifacts:
    paths:
      - docker-compose.prod.yml
  only:
    - main
    - merge_requests

# Запуск тестов
test:
  stage: test
  image: docker:latest
  services:
    - mysql:5.7
  script:
    - docker-compose -f docker-compose.ci.yml up -d
    - docker-compose -f docker-compose.ci.yml exec -T php-fpm composer test
    - docker-compose -f docker-compose.ci.yml down
  after_script:
    - docker-compose -f docker-compose.ci.yml down -v
  only:
    - merge_requests
    - main

# Деплой на staging
deploy-staging:
  stage: deploy
  image: alpine/ssh
  before_script:
    - apk add --no-cache openssh-client rsync
    - mkdir -p ~/.ssh
    - echo "$STAGING_SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
  script:
    - rsync -az --delete docker-compose.prod.yml .env.staging ${STAGING_USER}@${STAGING_HOST}:/app/${APP_NAME}/
    - ssh ${STAGING_USER}@${STAGING_HOST} "cd /app/${APP_NAME} && docker-compose -f docker-compose.prod.yml pull && docker-compose -f docker-compose.prod.yml up -d --force-recreate"
  environment:
    name: staging
    url: https://${STAGING_HOST}
  only:
    - main

# Деплой на production (ручной)
deploy-prod:
  stage: deploy
  image: alpine/ssh
  before_script:
    - apk add --no-cache openssh-client rsync
    - mkdir -p ~/.ssh
    - echo "$PROD_SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
  script:
    - rsync -az --delete docker-compose.prod.yml .env.prod ${PROD_USER}@${PRODUCTION_HOST}:/app/${APP_NAME}/
    - ssh ${PROD_USER}@${PRODUCTION_HOST} "cd /app/${APP_NAME} && docker-compose -f docker-compose.prod.yml pull && docker-compose -f docker-compose.prod.yml up -d --force-recreate"
  environment:
    name: production
    url: https://${PRODUCTION_HOST}
  when: manual
  only:
    - tags
```

## Необходимые файлы конфигурации

### 1. `docker-compose.ci.yml` (для CI)

```yaml
version: '3.8'

services:
  php-fpm:
    build:
      context: .
      target: ci
    image: ${CI_REGISTRY_IMAGE}/php-fpm:${CI_COMMIT_SHORT_SHA}
    environment:
      - DB_HOST=mysql
      - DB_USER=user
      - DB_PASSWORD=password
      - DB_NAME=test_db
    volumes:
      - ./:/var/www/html
    depends_on:
      - mysql

  nginx:
    image: nginx:alpine
    ports:
      - "8000:80"
    volumes:
      - ./docker/nginx.conf:/etc/nginx/conf.d/default.conf
      - ./:/var/www/html
    depends_on:
      - php-fpm

  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: test_db
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    ports:
      - "3306:3306"
```

### 2. `docker-compose.prod.yml` (для продакшена)

```yaml
version: '3.8'

services:
  php-fpm:
    image: ${CI_REGISTRY_IMAGE}/php-fpm:${CI_COMMIT_SHORT_SHA}
    restart: always
    environment:
      - APP_ENV=prod
      - APP_DEBUG=0
      - DB_HOST=mysql
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_NAME=${DB_NAME}
    volumes:
      - php-data:/var/www/html/var

  nginx:
    image: nginx:alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx.conf:/etc/nginx/conf.d/default.conf
      - php-data:/var/www/html
      - ./docker/ssl:/etc/nginx/ssl
    depends_on:
      - php-fpm

  mysql:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mysql-data:/var/lib/mysql
    ports:
      - "3306:3306"

volumes:
  php-data:
  mysql-data:
```

### 3. `Dockerfile` для PHP

```dockerfile
# Этап для CI (с dev-зависимостями)
FROM composer:2 as ci

WORKDIR /app
COPY . .
RUN composer install --prefer-dist --no-ansi --no-interaction --no-progress

# Этап для production
FROM php:8.2-fpm-alpine

# Установка зависимостей
RUN apk add --no-cache \
    nginx \
    supervisor \
    libzip-dev \
    && docker-php-ext-install zip pdo pdo_mysql opcache

# Настройка Nginx и PHP-FPM
COPY docker/nginx.conf /etc/nginx/conf.d/default.conf
COPY docker/supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY docker/php.ini /usr/local/etc/php/conf.d/custom.ini

# Копирование приложения
WORKDIR /var/www/html
COPY --from=ci /app .
RUN composer dump-autoload --optimize --no-dev --classmap-authoritative

# Настройка прав
RUN chown -R www-data:www-data /var/www/html

EXPOSE 9000
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

## Настройка окружения

1. **Переменные GitLab CI/CD**:
   - `STAGING_SSH_PRIVATE_KEY` - приватный ключ для доступа к staging серверу
   - `PROD_SSH_PRIVATE_KEY` - приватный ключ для доступа к production серверу
   - `STAGING_USER`, `PROD_USER` - пользователи для SSH
   - `DB_*` переменные для подключения к БД

2. **Настройка серверов**:
   - Установите Docker и Docker Compose на целевые серверы
   - Создайте директорию `/app/my-php-app` на серверах
   - Добавьте публичные ключи в `~/.ssh/authorized_keys`

3. **Дополнительные файлы**:
   - `docker/nginx.conf` - конфигурация Nginx
   - `docker/supervisord.conf` - конфигурация Supervisor
   - `docker/php.ini` - настройки PHP

Эта конфигурация обеспечит полный цикл CI/CD для PHP-приложения с использованием Docker Compose, включая линтинг, тестирование и деплой на staging/production окружения.