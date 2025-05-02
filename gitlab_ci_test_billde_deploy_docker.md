# Настройка GitLab CI/CD для сборки, тестирования и деплоя Docker

Вот пример настройки `.gitlab-ci.yml` для автоматической сборки Docker-образа, его тестирования и деплоя.

## Базовый пример `.gitlab-ci.yml`

```yaml
stages:
  - build
  - test
  - deploy

variables:
  # Имя Docker-образа
  IMAGE_NAME: my-app
  # Тег образа (используем commit SHA)
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA
  # Реестр Docker (можно использовать GitLab Registry)
  DOCKER_REGISTRY: $CI_REGISTRY
  # Полный путь к образу
  IMAGE_FULL: $DOCKER_REGISTRY/$CI_PROJECT_PATH/$IMAGE_NAME:$IMAGE_TAG

# Используем Docker-in-Docker (dind) сервис
services:
  - docker:dind

before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

build:
  stage: build
  script:
    - docker build -t $IMAGE_FULL .
    - docker push $IMAGE_FULL
  only:
    - main
    - merge_requests

test:
  stage: test
  script:
    - docker run $IMAGE_FULL npm test  # пример для Node.js приложения
  only:
    - main
    - merge_requests

deploy:
  stage: deploy
  script:
    - echo "Deploying to production..."
    # Здесь могут быть команды для деплоя, например:
    # - kubectl set image deployment/my-app my-app=$IMAGE_FULL
    # или
    # - ssh user@server "docker pull $IMAGE_FULL && docker-compose up -d"
  environment:
    name: production
    url: https://your-production-url.com
  only:
    - main
  when: manual  # Деплой требует ручного подтверждения
```

## Расширенный пример с кэшированием и дополнительными этапами

```yaml
stages:
  - lint
  - build
  - test
  - deploy-staging
  - deploy-prod

variables:
  IMAGE_NAME: my-app
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA
  DOCKER_REGISTRY: $CI_REGISTRY
  IMAGE_FULL: $DOCKER_REGISTRY/$CI_PROJECT_PATH/$IMAGE_NAME:$IMAGE_TAG
  IMAGE_LATEST: $DOCKER_REGISTRY/$CI_PROJECT_PATH/$IMAGE_NAME:latest

services:
  - docker:dind

before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

lint:
  stage: lint
  image: docker:latest
  script:
    - docker run --rm -v "$(pwd):/app" hadolint/hadolint hadolint /app/Dockerfile
  only:
    - merge_requests

build:
  stage: build
  script:
    - docker build --pull -t $IMAGE_FULL .
    - docker tag $IMAGE_FULL $IMAGE_LATEST
    - docker push $IMAGE_FULL
    - docker push $IMAGE_LATEST
  cache:
    key: docker-cache
    paths:
      - .docker-cache/
    policy: pull
  artifacts:
    paths:
      - docker-compose.yml
      - configs/
  only:
    - main
    - merge_requests

unit-test:
  stage: test
  script:
    - docker run --rm $IMAGE_FULL npm run test:unit

integration-test:
  stage: test
  script:
    - docker-compose -f docker-compose.test.yml up -d
    - docker run --network container:my-app-test alpine/nc -z localhost 3000
    - docker-compose -f docker-compose.test.yml down

deploy-staging:
  stage: deploy-staging
  script:
    - echo "Deploying to staging environment..."
    - kubectl config use-context staging
    - kubectl set image deployment/my-app my-app=$IMAGE_FULL
  environment:
    name: staging
    url: https://staging.your-app.com
  only:
    - main

deploy-prod:
  stage: deploy-prod
  script:
    - echo "Deploying to production environment..."
    - kubectl config use-context production
    - kubectl set image deployment/my-app my-app=$IMAGE_FULL
  environment:
    name: production
    url: https://your-app.com
  when: manual
  only:
    - tags
```

## Необходимые настройки в GitLab

1. **Переменные окружения**:
   - Зайдите в `Settings > CI/CD > Variables`
   - Добавьте необходимые переменные (например, учетные данные для доступа к вашему реестру или кластеру Kubernetes)

2. **Docker Registry**:
   - GitLab предоставляет встроенный Docker Registry, который можно использовать
   - Для внешнего реестра нужно добавить соответствующие переменные (`DOCKER_REGISTRY`, `DOCKER_USER`, `DOCKER_PASSWORD`)

3. **Runner**:
   - Убедитесь, что у вас настроен Runner с поддержкой Docker (обычно используется `docker-in-docker`)

## Советы по оптимизации

1. Используйте многоэтапные сборки в Dockerfile для уменьшения размера конечного образа
2. Настройте кэширование слоев Docker для ускорения сборки
3. Для больших проектов рассмотрите использование BuildKit
4. Добавьте уведомления о статусе сборки (Slack, email и т.д.)

Пример оптимизированного Dockerfile:

```dockerfile
# Этап сборки
FROM node:16 as builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Этап выполнения
FROM node:16-alpine
WORKDIR /app
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

Это базовая конфигурация, которую можно адаптировать под конкретные нужды вашего проекта.