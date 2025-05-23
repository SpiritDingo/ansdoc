# Роль Ansible для установки и настройки Jenkins в Docker контейнере с использованием Docker Compose

Вот пример роли Ansible для развертывания Jenkins в Docker контейнере с использованием Docker Compose.

## Структура роли

```
roles/jenkins-docker/
├── defaults
│   └── main.yml
├── files
│   └── jenkins-compose.yml
├── tasks
│   └── main.yml
├── templates
│   └── jenkins.env.j2
└── vars
    └── main.yml
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Имя сервиса Jenkins в Docker Compose
jenkins_service_name: jenkins

# Порт для доступа к Jenkins
jenkins_host_port: 8080

# Порт для агентов Jenkins
jenkins_agent_port: 50000

# Путь для хранения данных Jenkins
jenkins_data_path: /var/jenkins_home

# Версия образа Jenkins
jenkins_image_version: "lts"

# Имя пользователя администратора Jenkins
jenkins_admin_user: admin

# Имя Docker сети
jenkins_network_name: jenkins-net
```

### vars/main.yml

```yaml
---
# Пароль администратора Jenkins (должен быть переопределен в групповых переменных или vault)
jenkins_admin_password: "changeme"
```

### files/jenkins-compose.yml

```yaml
version: '3.8'

services:
  {{ jenkins_service_name }}:
    image: jenkins/jenkins:{{ jenkins_image_version }}
    container_name: {{ jenkins_service_name }}
    restart: unless-stopped
    ports:
      - "{{ jenkins_host_port }}:8080"
      - "{{ jenkins_agent_port }}:50000"
    volumes:
      - {{ jenkins_data_path }}:/var/jenkins_home
    env_file:
      - jenkins.env
    networks:
      - {{ jenkins_network_name }}

networks:
  {{ jenkins_network_name }}:
    driver: bridge
```

### templates/jenkins.env.j2

```
JENKINS_ADMIN_ID={{ jenkins_admin_user }}
JENKINS_ADMIN_PASSWORD={{ jenkins_admin_password }}
```

### tasks/main.yml

```yaml
---
- name: Установка зависимостей
  apt:
    name:
      - docker.io
      - docker-compose
    state: present
    update_cache: yes
  when: ansible_os_family == 'Debian'

- name: Установка зависимостей (для RHEL)
  yum:
    name:
      - docker
      - docker-compose
    state: present
  when: ansible_os_family == 'RedHat'

- name: Запуск и добавление в автозагрузку Docker
  service:
    name: docker
    state: started
    enabled: yes

- name: Создание директории для данных Jenkins
  file:
    path: "{{ jenkins_data_path }}"
    state: directory
    mode: '0755'

- name: Копирование файла окружения Jenkins
  template:
    src: jenkins.env.j2
    dest: "{{ jenkins_data_path }}/jenkins.env"
    mode: '0600'

- name: Копирование docker-compose файла
  copy:
    src: jenkins-compose.yml
    dest: "{{ jenkins_data_path }}/docker-compose.yml"
    mode: '0644'

- name: Запуск Jenkins в Docker
  command: docker-compose up -d
  args:
    chdir: "{{ jenkins_data_path }}"
  register: docker_compose_result
  changed_when: "'Creating' in docker_compose.stdout or 'Starting' in docker_compose.stdout"

- name: Ожидание запуска Jenkins
  uri:
    url: "http://localhost:{{ jenkins_host_port }}"
    status_code: 200
    timeout: 30
  register: jenkins_status
  retries: 10
  delay: 10
  until: jenkins_status.status == 200
  ignore_errors: yes
```

## Использование роли

1. Создайте playbook (например, `jenkins.yml`):

```yaml
---
- hosts: jenkins_servers
  become: yes
  roles:
    - jenkins-docker
```

2. Создайте файл `group_vars/jenkins_servers.yml` с переопределением переменных:

```yaml
---
jenkins_admin_password: "secure_password_here"
jenkins_host_port: 8081  # если нужно изменить порт
```

3. Запустите playbook:

```bash
ansible-playbook -i inventory jenkins.yml
```

## Дополнительные улучшения

1. Для безопасности рекомендуется:
   - Хранить пароли в Ansible Vault
   - Добавить настройку TLS/SSL
   - Настроить брандмауэр

2. Можно добавить задачи для:
   - Установки плагинов Jenkins
   - Настройки начальных job
   - Настройки backup

3. Для production рекомендуется использовать отдельные volumes для данных.