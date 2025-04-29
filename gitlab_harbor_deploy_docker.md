# Настройка GitLab CI/CD с Harbor для сборки, хранения Docker образов и деплоя на сервер

## Полная схема работы
1. GitLab Runner собирает Docker образ из кода
2. Образ сохраняется в Harbor (приватный Docker Registry)
3. Образ развертывается на целевом Docker сервере

## 1. Предварительные требования

- Установленный GitLab (CE/EE или GitLab.com)
- Настроенный Harbor (v2.0+)
- Docker сервер для деплоя с SSH доступом
- GitLab Runner с поддержкой Docker

## 2. Настройка переменных в GitLab

В настройках проекта (`Settings > CI/CD > Variables`) добавьте:

```
HARBOR_URL = https://harbor.your-domain.com
HARBOR_USER = ci-user
HARBOR_PASSWORD = strong-password (masked)
HARBOR_PROJECT = your-project

DEPLOY_SERVER = server-ip
DEPLOY_SSH_USER = deploy-user
DEPLOY_SSH_KEY = приватный SSH ключ (masked, file type)
```

## 3. Полный пример .gitlab-ci.yml

```yaml
stages:
  - build
  - test
  - deploy

variables:
  IMAGE_NAME: $HARBOR_URL/$HARBOR_PROJECT/my-app
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA

before_script:
  - echo "$HARBOR_PASSWORD" | docker login -u "$HARBOR_USER" --password-stdin $HARBOR_URL

build:
  stage: build
  script:
    - docker build -t $IMAGE_NAME:$IMAGE_TAG .
    - docker push $IMAGE_NAME:$IMAGE_TAG
  only:
    - main
    - master
    - tags

test:
  stage: test
  script:
    - docker pull $IMAGE_NAME:$IMAGE_TAG
    - docker run --rm $IMAGE_NAME:$IMAGE_TAG npm test

deploy:
  stage: deploy
  before_script:
    - mkdir -p ~/.ssh
    - echo "$DEPLOY_SSH_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh-keyscan -H $DEPLOY_SERVER >> ~/.ssh/known_hosts
  script:
    - ssh $DEPLOY_SSH_USER@$DEPLOY_SERVER "
        echo '$HARBOR_PASSWORD' | docker login -u '$HARBOR_USER' --password-stdin $HARBOR_URL &&
        docker stop my-app || true &&
        docker rm my-app || true &&
        docker pull $IMAGE_NAME:$IMAGE_TAG &&
        docker run -d --name my-app -p 3000:3000 $IMAGE_NAME:$IMAGE_TAG
      "
  only:
    - main
    - master
```

## 4. Дополнительные настройки

### 4.1. Переменные окружения для приложения

Добавьте в команду docker run:

```yaml
docker run -d --name my-app -p 3000:3000 -e "DB_HOST=db.example.com" $IMAGE_NAME:$IMAGE_TAG
```

### 4.2. Использование Docker Compose на сервере

1. Создайте файл `docker-compose.yml` в вашем проекте
2. Модифицируйте этап deploy:

```yaml
deploy:
  stage: deploy
  script:
    - scp docker-compose.yml $DEPLOY_SSH_USER@$DEPLOY_SERVER:~/my-app/
    - ssh $DEPLOY_SSH_USER@$DEPLOY_SERVER "
        cd ~/my-app &&
        echo '$HARBOR_PASSWORD' | docker login -u '$HARBOR_USER' --password-stdin $HARBOR_URL &&
        docker-compose pull &&
        docker-compose up -d
      "
```

### 4.3. Health check и автоматический откат

```yaml
deploy:
  # ...
  script:
    - ssh $DEPLOY_SSH_USER@$DEPLOY_SERVER "
        # ... предыдущие команды ...
        sleep 10
        if curl -s http://localhost:3000/health | grep -q 'OK'; then
          echo 'Deployment successful'
        else
          echo 'Deployment failed, rolling back'
          docker stop my-app && docker rm my-app
          docker run -d --name my-app -p 3000:3000 $IMAGE_NAME:previous-stable-tag
          exit 1
        fi
      "
```

## 5. Настройка сервера для деплоя

1. Установите Docker на целевой сервер:
```bash
curl -fsSL https://get.docker.com | sh
```

2. Добавьте пользователя для деплоя в группу docker:
```bash
usermod -aG docker $DEPLOY_SSH_USER
```

3. Настройте SSH доступ по ключу

## 6. Рекомендации по безопасности

1. Используйте Robot Accounts в Harbor с ограниченными правами
2. Ограничьте доступ к серверу только с IP GitLab Runner
3. Используйте сети Docker и ограничивайте экспорт портов
4. Регулярно обновляйте образы и сканируйте на уязвимости
5. Для production используйте секреты (Docker Secrets или HashiCorp Vault)

## 7. Альтернативный вариант - использование GitLab Deployment

Для Kubernetes или других платформ можно использовать встроенные возможности GitLab:

```yaml
deploy:
  stage: deploy
  image: bitnami/kubectl
  script:
    - kubectl set image deployment/my-app my-app=$IMAGE_NAME:$IMAGE_TAG
  environment:
    name: production
    url: https://your-app.com
```

## Заключение

Данная конфигурация обеспечивает полный цикл:
- Сборка Docker образа при коммите
- Хранение в Harbor
- Автоматический деплой на целевой сервер
- Откат при неудачном деплое

Вы можете расширить pipeline, добавив этапы тестирования, проверки безопасности или деплоя в несколько окружений.