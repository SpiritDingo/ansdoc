# Настройка GitLab CI/CD для PHP-проекта с Docker

Вот подробная конфигурация `.gitlab-ci.yml` для PHP-приложения, включающая сборку Docker-образа, тестирование и деплой.

## Полный пример `.gitlab-ci.yml` для PHP

```yaml
stages:
  - lint
  - build
  - test
  - deploy

variables:
  # Настройки Docker
  DOCKER_HOST: tcp://docker:2375
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""
  
  # Настройки образа
  IMAGE_NAME: php-app
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA
  DOCKER_REGISTRY: $CI_REGISTRY
  IMAGE_FULL: $DOCKER_REGISTRY/$CI_PROJECT_PATH/$IMAGE_NAME:$IMAGE_TAG
  IMAGE_LATEST: $DOCKER_REGISTRY/$CI_PROJECT_PATH/$IMAGE_NAME:latest
  
  # Настройки PHP
  COMPOSER_ALLOW_SUPERUSER: 1
  COMPOSER_CACHE_DIR: /tmp/composer-cache

services:
  - docker:dind

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - vendor/
    - $COMPOSER_CACHE_DIR

before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

## Этапы CI/CD

# Линтинг кода
lint:
  stage: lint
  image: composer:latest
  script:
    - composer install --prefer-dist --no-ansi --no-interaction --no-progress --no-scripts
    - vendor/bin/phpcs --standard=PSR12 src/
    - vendor/bin/phpstan analyse --level=8 src/
  only:
    - merge_requests

# Сборка Docker-образа
build:
  stage: build
  script:
    - docker build --pull -t $IMAGE_FULL .
    - docker tag $IMAGE_FULL $IMAGE_LATEST
    - docker push $IMAGE_FULL
    - docker push $IMAGE_LATEST
  artifacts:
    paths:
      - docker-compose.yml
  only:
    - main
    - merge_requests

# Запуск тестов
test:
  stage: test
  script:
    - docker run --rm $IMAGE_FULL vendor/bin/phpunit --testsuite unit
    - docker run --rm $IMAGE_FULL vendor/bin/phpunit --testsuite integration
  coverage: '/^\s*Lines:\s*\d+.\d+\%/'
  allow_failure: false

# Деплой на staging
deploy-staging:
  stage: deploy
  script:
    - echo "Deploying to staging..."
    - kubectl config use-context staging
    - kubectl apply -f k8s/staging/
    - kubectl set image deployment/php-app php-app=$IMAGE_FULL
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - main

# Деплой на production (ручной)
deploy-prod:
  stage: deploy
  script:
    - echo "Deploying to production..."
    - kubectl config use-context production
    - kubectl apply -f k8s/production/
    - kubectl set image deployment/php-app php-app=$IMAGE_FULL
  environment:
    name: production
    url: https://example.com
  when: manual
  only:
    - tags
```

## Пример Dockerfile для PHP-приложения

```dockerfile
# Этап сборки
FROM composer:2 as builder

WORKDIR /app
COPY . .
RUN composer install \
    --prefer-dist \
    --no-dev \
    --no-scripts \
    --no-interaction \
    --optimize-autoloader

# Этап выполнения
FROM php:8.2-fpm-alpine

# Установка зависимостей
RUN apk add --no-cache \
    nginx \
    supervisor \
    libzip-dev \
    && docker-php-ext-install zip pdo pdo_mysql opcache

# Настройка Nginx и PHP-FPM
COPY docker/nginx.conf /etc/nginx/nginx.conf
COPY docker/supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY docker/php.ini /usr/local/etc/php/conf.d/custom.ini

# Копирование приложения
WORKDIR /var/www/html
COPY --from=builder /app .
COPY . .

# Настройка прав
RUN chown -R www-data:www-data /var/www/html

EXPOSE 80
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

## Дополнительные настройки

### 1. Настройка Kubernetes (пример для deploy stages)

Создайте файлы в директории `k8s/`:

`k8s/staging/deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: php-app
  template:
    metadata:
      labels:
        app: php-app
    spec:
      containers:
      - name: php-app
        image: $IMAGE_FULL
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: php-app-config
```

### 2. Настройка Composer для кэширования

Добавьте в `composer.json`:
```json
{
  "config": {
    "cache-dir": "/tmp/composer-cache",
    "sort-packages": true
  }
}
```

### 3. Настройка тестов PHPUnit

Пример `phpunit.xml.dist`:
```xml
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.3/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         colors="true">
    <testsuites>
        <testsuite name="unit">
            <directory>tests/Unit</directory>
        </testsuite>
        <testsuite name="integration">
            <directory>tests/Integration</directory>
        </testsuite>
    </testsuites>
    <coverage>
        <include>
            <directory>src</directory>
        </include>
    </coverage>
</phpunit>
```

## Особенности для PHP-проектов

1. **Кэширование зависимостей Composer**:
   - Используем кэш GitLab CI для ускорения установки зависимостей
   - Сохраняем vendor между этапами

2. **Многоэтапная сборка Docker**:
   - Отдельный этап для установки зависимостей Composer
   - Финальный образ без dev-зависимостей

3. **Тестирование**:
   - PHPUnit для unit и integration тестов
   - PHPStan для статического анализа
   - PHPCS для проверки стиля кода

4. **Деплой**:
   - Варианты для Kubernetes или обычных серверов
   - Разделение на staging и production

Эта конфигурация обеспечит полный цикл CI/CD для вашего PHP-приложения с использованием Docker.