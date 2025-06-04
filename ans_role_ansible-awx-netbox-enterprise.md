# Enterprise-решение для Ansible, AWX и Netbox в Docker: расширенная настройка

Вот усовершенствованная версия роли для корпоративного развертывания с высокой доступностью, безопасностью и мониторингом.

## Улучшения для enterprise-среды:

1. **Кластеризация и отказоустойчивость**
2. **Расширенная безопасность**
3. **Мониторинг и логирование**
4. **Резервное копирование**
5. **Интеграция с корпоративными системами**

## Обновленная структура роли

```
ansible-awx-netbox-enterprise/
├── defaults/
│   └── main.yml
├── files/
│   ├── docker-compose/
│   │   ├── awx-cluster.yml
│   │   ├── netbox-ha.yml
│   │   ├── monitoring.yml
│   │   └── backup.yml
│   ├── configs/
│   │   ├── awx/
│   │   │   ├── settings.py
│   │   │   └── nginx/
│   │   ├── netbox/
│   │   │   ├── configuration.py
│   │   │   ├── ldap_config.py
│   │   │   └── logging.yml
│   │   └── prometheus/
│   ├── scripts/
│   │   ├── backup/
│   │   ├── restore/
│   │   └── healthchecks/
│   └── certs/
│       ├── generate_certs.sh
│       └── templates/
├── tasks/
│   ├── main.yml
│   ├── security.yml
│   ├── monitoring.yml
│   └── backup.yml
├── templates/
│   ├── netbox.env.j2
│   ├── awx.env.j2
│   ├── ldap.env.j2
│   └── prometheus/
└── vars/
    ├── main.yml
    └── enterprise.yml
```

## Ключевые файлы конфигурации (выборочно)

### files/docker-compose/awx-cluster.yml

```yaml
version: '3.8'

services:
  awx-postgres:
    image: postgres:13
    deploy:
      replicas: 2
      placement:
        constraints: [node.role == manager]
    configs:
      - source: postgres_config
        target: /etc/postgresql/postgresql.conf
    secrets:
      - postgres_password
    volumes:
      - awx_postgres_data:/var/lib/postgresql/data
    networks:
      - awx_backend

  awx-rabbitmq:
    image: rabbitmq:3.8-management-alpine
    deploy:
      replicas: 3
    environment:
      RABBITMQ_ERLANG_COOKIE: ${RABBITMQ_ERLANG_COOKIE}
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_DEFAULT_USER}
    secrets:
      - rabbitmq_password
    volumes:
      - awx_rabbitmq_data:/var/lib/rabbitmq
    networks:
      - awx_backend

  awx-task:
    image: ansible/awx:${AWX_VERSION}
    deploy:
      replicas: 3
    environment:
      AWX_ADMIN_USER: ${AWX_ADMIN_USER}
    secrets:
      - awx_admin_password
      - awx_secret_key
      - postgres_password
      - rabbitmq_password
    configs:
      - source: awx_settings
        target: /etc/awx/settings.py
    volumes:
      - awx_projects:/var/lib/awx/projects
    networks:
      - awx_backend
      - awx_frontend

  awx-web:
    image: ansible/awx:${AWX_VERSION}
    deploy:
      replicas: 3
      labels:
        - traefik.enable=true
        - traefik.http.routers.awx.rule=Host(`awx.company.com`)
        - traefik.http.routers.awx.tls=true
        - traefik.http.routers.awx.tls.certresolver=le
    secrets:
      - awx_secret_key
    depends_on:
      - awx-task
    networks:
      - awx_frontend
      - awx_backend

networks:
  awx_backend:
    driver: overlay
    internal: true
  awx_frontend:
    driver: overlay

volumes:
  awx_postgres_data:
    driver: glusterfs
    driver_opts:
      volume: awx_postgres
  awx_rabbitmq_data:
    driver: glusterfs
    driver_opts:
      volume: awx_rabbitmq
  awx_projects:
    driver: glusterfs
    driver_opts:
      volume: awx_projects

configs:
  postgres_config:
    file: ./configs/postgres/postgresql.conf
  awx_settings:
    file: ./configs/awx/settings.py

secrets:
  postgres_password:
    external: true
  rabbitmq_password:
    external: true
  awx_admin_password:
    external: true
  awx_secret_key:
    external: true
```

### files/configs/netbox/configuration.py (фрагмент)

```python
import os
from netbox.configuration import Configuration

class EnterpriseConfiguration(Configuration):
    # Кластерная конфигурация
    DATABASE = {
        'NAME': os.getenv('NETBOX_DB_NAME', 'netbox'),
        'USER': os.getenv('NETBOX_DB_USER', 'netbox'),
        'PASSWORD': os.getenv('NETBOX_DB_PASSWORD', ''),
        'HOST': os.getenv('NETBOX_DB_HOST', 'postgres-cluster'),
        'PORT': os.getenv('NETBOX_DB_PORT', '5432'),
        'CONN_MAX_AGE': 300,
        'OPTIONS': {
            'sslmode': 'require',
            'sslrootcert': '/etc/netbox/config/db-ca.pem'
        }
    }

    # Настройки Redis для кластера
    REDIS = {
        'tasks': {
            'HOST': 'redis-cluster',
            'PORT': 6379,
            'PASSWORD': os.getenv('REDIS_PASSWORD', ''),
            'DATABASE': 0,
            'SSL': True,
            'DEFAULT_TIMEOUT': 300,
            'CACHE_TIMEOUT': 60,
        },
        'caching': {
            'HOST': 'redis-cluster',
            'PORT': 6379,
            'PASSWORD': os.getenv('REDIS_PASSWORD', ''),
            'DATABASE': 1,
            'SSL': True,
        }
    }

    # Безопасность
    CSRF_TRUSTED_ORIGINS = [
        'https://netbox.company.com',
        'https://awx.company.com'
    ]
    ALLOWED_HOSTS = ['netbox.company.com']
    SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
    SECURE_SSL_REDIRECT = True
    SESSION_COOKIE_SECURE = True
    CSRF_COOKIE_SECURE = True
    SECURE_HSTS_SECONDS = 31536000
    SECURE_HSTS_INCLUDE_SUBDOMAINS = True
    SECURE_HSTS_PRELOAD = True
    SECURE_CONTENT_TYPE_NOSNIFF = True
    X_FRAME_OPTIONS = 'DENY'

    # LDAP интеграция
    AUTHENTICATION_BACKENDS = (
        'netbox.authentication.LDAPBackend',
        'django.contrib.auth.backends.ModelBackend',
    )

    # Расширенное логирование
    LOGGING = {
        'version': 1,
        'disable_existing_loggers': False,
        'formatters': {
            'verbose': {
                'format': '{levelname} {asctime} {module} {message}',
                'style': '{',
            },
        },
        'handlers': {
            'file': {
                'level': 'DEBUG',
                'class': 'logging.handlers.RotatingFileHandler',
                'filename': '/var/log/netbox/netbox.log',
                'maxBytes': 1024*1024*10, # 10 MB
                'backupCount': 10,
                'formatter': 'verbose',
            },
            'logstash': {
                'level': 'INFO',
                'class': 'logstash.TCPLogstashHandler',
                'host': 'logstash',
                'port': 5000,
                'version': 1,
                'tags': ['netbox'],
            },
        },
        'loggers': {
            'django': {
                'handlers': ['file', 'logstash'],
                'level': 'INFO',
            },
            'netbox': {
                'handlers': ['file', 'logstash'],
                'level': 'DEBUG',
            },
        },
    }

config = EnterpriseConfiguration()
```

### tasks/security.yml

```yaml
- name: Установка зависимостей для безопасности
  apt:
    name:
      - fail2ban
      - auditd
      - libpam-pwquality
    state: present

- name: Настройка базовых параметров безопасности
  template:
    src: templates/sysctl_security.j2
    dest: /etc/sysctl.d/99-security.conf
    mode: 0644
  notify: Apply sysctl settings

- name: Настройка Docker daemon security
  copy:
    content: |
      {
        "userns-remap": "default",
        "log-driver": "json-file",
        "log-opts": {
          "max-size": "10m",
          "max-file": "3"
        },
        "default-ulimits": {
          "nofile": {
            "Name": "nofile",
            "Hard": 65536,
            "Soft": 65536
          }
        },
        "icc": false,
        "live-restore": true
      }
    dest: /etc/docker/daemon.json
    mode: 0644
  notify: Restart docker

- name: Создание сетей Docker с ограничениями
  community.docker.docker_network:
    name: "{{ item.name }}"
    driver: overlay
    ipam_options:
      driver: default
      config:
        - subnet: "{{ item.subnet }}"
    attachable: false
    internal: "{{ item.internal | default(false) }}"
    state: present
  loop:
    - { name: awx_internal, subnet: "10.10.10.0/24", internal: true }
    - { name: netbox_internal, subnet: "10.10.20.0/24", internal: true }
    - { name: monitoring, subnet: "10.10.30.0/24" }

- name: Создание Docker secrets
  community.docker.docker_secret:
    name: "{{ item.name }}"
    data: "{{ item.value }}"
    labels:
      project: "ansible-awx-netbox"
      environment: "production"
  loop:
    - { name: postgres_password, value: "{{ lookup('password', '/dev/null length=32 chars=ascii_letters,digits') }}" }
    - { name: rabbitmq_password, value: "{{ lookup('password', '/dev/null length=32 chars=ascii_letters,digits') }}" }
    - { name: awx_admin_password, value: "{{ awx_admin_password }}" }
    - { name: awx_secret_key, value: "{{ lookup('password', '/dev/null length=64 chars=ascii_letters,digits') }}" }
    - { name: netbox_secret_key, value: "{{ lookup('password', '/dev/null length=64 chars=ascii_letters,digits') }}" }
  no_log: true

- name: Настройка TLS сертификатов
  copy:
    src: files/certs/
    dest: /opt/certs/
    mode: 0600
  when: inventory_hostname in groups['cert_manager']

- name: Генерация сертификатов
  command: /opt/certs/generate_certs.sh
  args:
    chdir: /opt/certs
  when: inventory_hostname in groups['cert_manager']
```

### tasks/monitoring.yml

```yaml
- name: Развертывание стека мониторинга
  community.docker.docker_stack:
    name: monitoring
    state: present
    compose:
      - /opt/docker-compose/monitoring.yml

- name: Настройка Prometheus для AWX
  uri:
    url: "http://localhost:9090/api/v1/admin/tsdb/snapshot"
    method: POST
    body_format: json
    body:
      name: "awx_metrics"
      interval: "1m"
    status_code: 200
    register: prometheus_snapshot
  retries: 5
  delay: 10
  until: prometheus_snapshot.status == 200

- name: Настройка Grafana dashboards
  template:
    src: "templates/grafana/{{ item }}.json.j2"
    dest: "/opt/grafana/dashboards/{{ item }}.json"
    mode: 0644
  loop:
    - awx_monitoring
    - netbox_metrics
    - docker_health

- name: Импорт дашбордов в Grafana
  command: >
    curl -X POST -H "Content-Type: application/json" -H "Authorization: Bearer {{ grafana_api_token }}"
    -d @/opt/grafana/dashboards/{{ item }}.json
    http://grafana:3000/api/dashboards/db
  args:
    chdir: /opt/grafana/dashboards
  loop:
    - awx_monitoring
    - netbox_metrics
    - docker_health
  retries: 3
  delay: 5
```

## Интеграционные особенности для enterprise

1. **LDAP/Active Directory интеграция**:
   ```python
   # files/configs/netbox/ldap_config.py
   import ldap
   from django_auth_ldap.config import LDAPSearch, GroupOfNamesType

   AUTH_LDAP_SERVER_URI = "ldaps://ldap.company.com"
   AUTH_LDAP_BIND_DN = "CN=Netbox Service,OU=Service Accounts,DC=company,DC=com"
   AUTH_LDAP_BIND_PASSWORD = os.getenv('LDAP_BIND_PASSWORD')
   AUTH_LDAP_USER_SEARCH = LDAPSearch(
       "OU=Users,DC=company,DC=com",
       ldap.SCOPE_SUBTREE,
       "(sAMAccountName=%(user)s)"
   )
   AUTH_LDAP_GROUP_SEARCH = LDAPSearch(
       "OU=Groups,DC=company,DC=com",
       ldap.SCOPE_SUBTREE,
       "(objectClass=group)"
   )
   ```

2. **Интеграция с SIEM системами**:
   ```yaml
   # Настройка Filebeat для отправки логов в Splunk/ELK
   filebeat:
     image: docker.elastic.co/beats/filebeat:8.3.3
     volumes:
       - /var/lib/docker/containers:/var/lib/docker/containers:ro
       - /var/run/docker.sock:/var/run/docker.sock:ro
     configs:
       - source: filebeat_config
         target: /usr/share/filebeat/filebeat.yml
     deploy:
       mode: global
   ```

3. **Интеграция с ITSM системами**:
   ```python
   # Настройки для ServiceNow интеграции
   SERVICE_NOW = {
       'INSTANCE': 'company.service-now.com',
       'API_USER': os.getenv('SNOW_API_USER'),
       'API_PASS': os.getenv('SNOW_API_PASS'),
       'CMDB_API_PATH': '/api/now/table/cmdb_ci',
       'INCIDENT_API_PATH': '/api/now/table/incident'
   }
   ```

## Развертывание в кластере Docker Swarm

```yaml
- name: Инициализация Docker Swarm
  community.docker.docker_swarm:
    state: present
    advertise_addr: "{{ ansible_default_ipv4.address }}"
    listen_addr: "0.0.0.0:2377"
    default_addr_pool:
      - base: "10.10.0.0/16"
        size: 24
  when: inventory_hostname in groups['swarm_managers']

- name: Развертывание стека AWX
  community.docker.docker_stack:
    name: awx
    state: present
    compose:
      - /opt/docker-compose/awx-cluster.yml
  when: inventory_hostname in groups['swarm_managers']

- name: Развертывание стека Netbox
  community.docker.docker_stack:
    name: netbox
    state: present
    compose:
      - /opt/docker-compose/netbox-ha.yml
  when: inventory_hostname in groups['swarm_managers']
```

## Рекомендации для production-развертывания

1. **Инфраструктура**:
   - Минимум 3 ноды для каждого сервиса в разных AZ
   - Shared storage (GlusterFS, Ceph) для данных
   - Выделенные сети для межсервисного взаимодействия

2. **Безопасность**:
   - Network policies для ограничения трафика
   - Pod security policies
   - Регулярный ротацый секретов
   - Аудит доступа

3. **Мониторинг**:
   - Prometheus + Grafana для метрик
   - ELK или Loki для логов
   - Alertmanager для уведомлений

4. **Резервное копирование**:
   ```yaml
   - name: Настройка ежедневного бэкапа
     cron:
       name: "Netbox and AWX backup"
       minute: "0"
       hour: "2"
       job: "/opt/scripts/backup/full_backup.sh >> /var/log/backup.log 2>&1"
       user: root
   ```

5. **Обновления**:
   - Blue/Green deployments для обновлений
   - Canary релизы для новых версий
   - Тестовый стенд для проверки обновлений

Это enterprise-решение обеспечивает высокую доступность, безопасность и интеграцию с корпоративными системами, что делает его подходящим для крупных организаций.