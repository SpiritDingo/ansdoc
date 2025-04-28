# Настройка CI/CD в GitLab с использованием Harbor для хранения Docker образов

## 1. Предварительные требования

- Установленный и настроенный GitLab (самостоятельно или GitLab.com)
- Установленный и настроенный Harbor (реестр Docker образов)
- Доступ к проекту в GitLab с правами на настройку CI/CD
- Docker и Docker Compose установленные на runner'ах

## 2. Настройка интеграции GitLab с Harbor

### 2.1. Добавление переменных окружения в GitLab

Перейдите в `Settings > CI/CD > Variables` вашего проекта и добавьте:

- `HARBOR_URL` - URL вашего Harbor (например, `https://harbor.example.com`)
- `HARBOR_USER` - имя пользователя Harbor
- `HARBOR_PASSWORD` - пароль пользователя (отметьте как masked)
- `HARBOR_PROJECT` - название проекта в Harbor (например, `myproject`)

### 2.2. Настройка аутентификации Docker в runner'ах

Добавьте в ваш `.gitlab-ci.yml` шаг аутентификации:

```yaml
before_script:
  - echo "$HARBOR_PASSWORD" | docker login -u "$HARBOR_USER" --password-stdin $HARBOR_URL
```

## 3. Пример .gitlab-ci.yml для сборки и публикации образов

```yaml
stages:
  - build
  - test
  - deploy

variables:
  # Имя образа в Harbor
  IMAGE_NAME: $HARBOR_URL/$HARBOR_PROJECT/my-app
  # Тег образа - используется коммит или тег
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA

build:
  stage: build
  script:
    - docker build -t $IMAGE_NAME:$IMAGE_TAG .
    - docker push $IMAGE_NAME:$IMAGE_TAG
  only:
    - master
    - tags

test:
  stage: test
  script:
    - docker run $IMAGE_NAME:$IMAGE_TAG /bin/sh -c "run-tests.sh"
  
deploy:
  stage: deploy
  script:
    - echo "Deploying image $IMAGE_NAME:$IMAGE_TAG"
    # Здесь могут быть команды для развертывания в вашем окружении
    - kubectl set image deployment/my-app my-app=$IMAGE_NAME:$IMAGE_TAG
  only:
    - master
```

## 4. Дополнительные настройки

### 4.1. Использование тегов для версионирования

Модифицируйте переменные для поддержки версионирования:

```yaml
variables:
  IMAGE_TAG: $CI_COMMIT_TAG
  IMAGE_TAG: latest if [$CI_COMMIT_TAG == ""]; then echo $CI_COMMIT_SHORT_SHA; fi
```

### 4.2. Сканирование образов на уязвимости

Добавьте этап сканирования:

```yaml
scan:
  stage: test
  script:
    - docker pull $IMAGE_NAME:$IMAGE_TAG
    - docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --exit-code 1 $IMAGE_NAME:$IMAGE_TAG
```

### 4.3. Очистка старых образов

Добавьте задание для очистки:

```yaml
cleanup:
  stage: deploy
  script:
    - |
      # Удаляем образы старше 30 дней
      HARBOR_API_URL=${HARBOR_URL}/api/v2.0
      curl -u "$HARBOR_USER:$HARBOR_PASSWORD" -X GET "$HARBOR_API_URL/projects/$HARBOR_PROJECT/repositories/my-app/artifacts?page_size=100" | jq -r '.[] | select(.pull_time < "'$(date -d '30 days ago' +%Y-%m-%d)'") | .digest' | while read digest; do
        curl -u "$HARBOR_USER:$HARBOR_PASSWORD" -X DELETE "$HARBOR_API_URL/projects/$HARBOR_PROJECT/repositories/my-app/artifacts/$digest"
      done
  only:
    - schedules
```

## 5. Настройка Harbor

1. Создайте проект в Harbor с соответствующими правами доступа
2. Настройте автоматическое сканирование на уязвимости (если требуется)
3. Настройте политики хранения (retention policies) для автоматической очистки старых образов

## 6. Безопасность

- Используйте Robot Accounts в Harbor вместо пользовательских учетных данных
- Ограничивайте права доступа к проекту в Harbor
- Регулярно обновляйте базовые образы и сканируйте на уязвимости
- Используйте подписывание образов (content trust) если требуется повышенная безопасность

## Заключение

Эта конфигурация предоставляет базовый рабочий CI/CD pipeline для сборки Docker образов, их хранения в Harbor и развертывания. Вы можете расширить его в соответствии с вашими потребностями, добавив дополнительные этапы тестирования, развертывания в разных окружениях или интеграции с другими инструментами.