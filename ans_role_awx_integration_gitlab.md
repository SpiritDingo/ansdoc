# Роль Ansible для установки и настройки Docker Compose с NGINX, AWX и интеграцией с GitLab

Создадим роль Ansible для развертывания стека Docker Compose, включающего NGINX как reverse proxy, AWX (Ansible Tower的开源版本) и его интеграцию с GitLab.

## Структура роли

```
roles/awx_docker_stack/
├── defaults
│   └── main.yml
├── files
│   ├── docker-compose.yml.j2
│   ├── nginx.conf.j2
│   └── awx.env.j2
├── handlers
│   └── main.yml
├── tasks
│   ├── main.yml
│   ├── install_docker.yml
│   ├── setup_nginx.yml
│   ├── setup_awx.yml
│   └── integrate_gitlab.yml
└── templates
    └── custom_awx_settings.py.j2
```

## Основные файлы роли

### defaults/main.yml

```yaml
# Общие настройки
awx_domain: "awx.example.com"
awx_admin_user: "admin"
awx_admin_password: "changeme"
awx_secret_key: "your-secret-key-here"

# Настройки Docker
docker_compose_version: "v2.24.5"

# Настройки NGINX
nginx_ssl_cert: "/etc/ssl/certs/nginx.crt"
nginx_ssl_key: "/etc/ssl/private/nginx.key"
nginx_ssl_enabled: true

# Настройки GitLab
gitlab_url: "https://gitlab.example.com"
gitlab_api_token: "your-gitlab-api-token"
gitlab_webhook_secret: "your-webhook-secret"
```

### tasks/main.yml

```yaml
- name: Include Docker installation tasks
  include_tasks: install_docker.yml

- name: Include NGINX setup tasks
  include_tasks: setup_nginx.yml

- name: Include AWX setup tasks
  include_tasks: setup_awx.yml

- name: Include GitLab integration tasks
  include_tasks: integrate_gitlab.yml
```

### tasks/install_docker.yml

```yaml
- name: Install dependencies
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - software-properties-common
      - python3-pip
    state: present
    update_cache: yes

- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: Add Docker repository
  apt_repository:
    repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable
    state: present

- name: Install Docker CE
  apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
    state: present
    update_cache: yes

- name: Install Docker Compose
  get_url:
    url: "https://github.com/docker/compose/releases/download/{{ docker_compose_version }}/docker-compose-linux-x86_64"
    dest: /usr/local/bin/docker-compose
    mode: '0755'

- name: Ensure Docker service is running
  service:
    name: docker
    state: started
    enabled: yes

- name: Add current user to docker group
  user:
    name: "{{ ansible_user }}"
    groups: docker
    append: yes
```

### tasks/setup_nginx.yml

```yaml
- name: Install NGINX
  apt:
    name: nginx
    state: present

- name: Create SSL certificate directory
  file:
    path: /etc/ssl/{certs,private}
    state: directory
    mode: '0755'
  when: nginx_ssl_enabled

- name: Generate self-signed SSL certificate
  command: |
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout {{ nginx_ssl_key }} -out {{ nginx_ssl_cert }} \
    -subj "/CN={{ awx_domain }}"
  args:
    creates: "{{ nginx_ssl_cert }}"
  when: nginx_ssl_enabled

- name: Configure NGINX as reverse proxy for AWX
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/sites-available/awx
    mode: '0644'
  notify: restart nginx

- name: Enable NGINX site
  file:
    src: /etc/nginx/sites-available/awx
    dest: /etc/nginx/sites-enabled/awx
    state: link
  notify: restart nginx

- name: Ensure default site is disabled
  file:
    path: /etc/nginx/sites-enabled/default
    state: absent
  notify: restart nginx
```

### files/nginx.conf.j2

```nginx
server {
    listen 80;
    server_name {{ awx_domain }};
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name {{ awx_domain }};

    ssl_certificate {{ nginx_ssl_cert }};
    ssl_certificate_key {{ nginx_ssl_key }};

    location / {
        proxy_pass http://localhost:8052;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    client_max_body_size 100m;
}
```

### tasks/setup_awx.yml

```yaml
- name: Create AWX directories
  file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - /opt/awx
    - /opt/awx/pgdocker

- name: Deploy Docker Compose file for AWX
  template:
    src: docker-compose.yml.j2
    dest: /opt/awx/docker-compose.yml
    mode: '0644'

- name: Create AWX environment file
  template:
    src: awx.env.j2
    dest: /opt/awx/awx.env
    mode: '0644'

- name: Pull AWX Docker images
  command: docker-compose -f /opt/awx/docker-compose.yml pull
  args:
    chdir: /opt/awx

- name: Start AWX containers
  command: docker-compose -f /opt/awx/docker-compose.yml up -d
  args:
    chdir: /opt/awx

- name: Wait for AWX to be ready
  uri:
    url: "http://localhost:8052"
    status_code: 200
    timeout: 10
  register: awx_ready
  until: awx_ready.status == 200
  retries: 30
  delay: 10
```

### files/docker-compose.yml.j2

```yaml
version: '3'
services:
  postgres:
    image: postgres:13
    volumes:
      - /opt/awx/pgdocker:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: awx
      POSTGRES_PASSWORD: awxpass
      POSTGRES_DB: awx
    restart: unless-stopped

  redis:
    image: redis
    restart: unless-stopped

  awx_task:
    image: ansible/awx:latest
    hostname: awx
    user: root
    volumes:
      - /opt/awx/projects:/var/lib/awx/projects:rw
    environment:
      - AWX_ADMIN_USER={{ awx_admin_user }}
      - AWX_ADMIN_PASSWORD={{ awx_admin_password }}
      - SECRET_KEY={{ awx_secret_key }}
      - DATABASE_USER=awx
      - DATABASE_PASSWORD=awxpass
      - DATABASE_HOST=postgres
      - DATABASE_PORT=5432
      - DATABASE_NAME=awx
      - RABBITMQ_USER=guest
      - RABBITMQ_PASSWORD=guest
      - RABBITMQ_HOST=rabbitmq
      - RABBITMQ_PORT=5672
      - RABBITMQ_VHOST=awx
      - MEMCACHED_HOST=memcached
      - MEMCACHED_PORT=11211
      - AWX_SKIP_MIGRATIONS=False
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  awx_web:
    image: ansible/awx:latest
    hostname: awxweb
    ports:
      - "8052:8052"
    user: root
    environment:
      - AWX_ADMIN_USER={{ awx_admin_user }}
      - AWX_ADMIN_PASSWORD={{ awx_admin_password }}
      - SECRET_KEY={{ awx_secret_key }}
      - DATABASE_USER=awx
      - DATABASE_PASSWORD=awxpass
      - DATABASE_HOST=postgres
      - DATABASE_PORT=5432
      - DATABASE_NAME=awx
      - RABBITMQ_USER=guest
      - RABBITMQ_PASSWORD=guest
      - RABBITMQ_HOST=rabbitmq
      - RABBITMQ_PORT=5672
      - RABBITMQ_VHOST=awx
      - MEMCACHED_HOST=memcached
      - MEMCACHED_PORT=11211
      - AWX_SKIP_MIGRATIONS=False
    depends_on:
      - awx_task
      - postgres
      - redis
    restart: unless-stopped
```

### tasks/integrate_gitlab.yml

```yaml
- name: Wait for AWX API to be ready
  uri:
    url: "http://localhost:8052/api/v2/ping/"
    method: GET
    status_code: 200
    timeout: 10
  register: awx_api_ready
  until: awx_api_ready.status == 200
  retries: 30
  delay: 10

- name: Create GitLab credential in AWX
  uri:
    url: "http://localhost:8052/api/v2/credentials/"
    method: POST
    headers:
      Content-Type: "application/json"
    body_format: json
    body:
      name: "GitLab API Token"
      credential_type: 12  # GitLab API Token type
      organization: 1
      inputs:
        api_token: "{{ gitlab_api_token }}"
    user: "{{ awx_admin_user }}"
    password: "{{ awx_admin_password }}"
    force_basic_auth: yes
    status_code: 201
  register: gitlab_credential

- name: Configure GitLab integration in AWX settings
  template:
    src: custom_awx_settings.py.j2
    dest: /opt/awx/awx/settings/custom.py
  notify: restart awx

- name: Create GitLab project in AWX
  uri:
    url: "http://localhost:8052/api/v2/projects/"
    method: POST
    headers:
      Content-Type: "application/json"
    body_format: json
    body:
      name: "GitLab Project"
      scm_type: "git"
      scm_url: "{{ gitlab_url }}/your-group/your-project.git"
      scm_credential: "{{ gitlab_credential.json.id }}"
      scm_update_on_launch: true
      organization: 1
    user: "{{ awx_admin_user }}"
    password: "{{ awx_admin_password }}"
    force_basic_auth: yes
    status_code: 201
```

### templates/custom_awx_settings.py.j2

```python
# GitLab integration settings
AWX_GITLAB_WEBHOOK_SECRET = '{{ gitlab_webhook_secret }}'

# Enable GitLab webhook receiver
GITLAB_WEBHOOK_URL = 'https://{{ awx_domain }}/api/v2/gitlab/'

# Additional settings
AWX_PROOT_ENABLED = False
AWX_PROOT_SHOW_PATHS = []
```

### handlers/main.yml

```yaml
- name: restart nginx
  service:
    name: nginx
    state: restarted

- name: restart awx
  command: docker-compose -f /opt/awx/docker-compose.yml restart awx_web awx_task
  args:
    chdir: /opt/awx
```

## Использование роли

1. Создайте playbook для использования роли:

```yaml
- name: Deploy AWX with Docker Compose and GitLab integration
  hosts: awx_server
  become: yes
  roles:
    - awx_docker_stack
```

2. Создайте файл inventory с вашими серверами:

```ini
[awx_server]
awx.example.com
```

3. Запустите playbook:

```bash
ansible-playbook -i inventory.ini deploy_awx.yml
```

## Дополнительные настройки

Для полной интеграции с GitLab вам также потребуется:

1. Настроить webhook в GitLab для автоматического обновления проектов в AWX при push-событиях.
2. Настроить CI/CD pipeline в GitLab для запуска playbook через AWX.
3. Настроить аутентификацию AWX через GitLab OAuth, если требуется.

Эта роль предоставляет базовую настройку стека AWX в Docker Compose с NGINX и интеграцией с GitLab. Вы можете дополнительно настроить её под свои требования.