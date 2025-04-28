# Установка и настройка Rundeck с помощью Ansible роли

## Создание структуры роли Ansible

Сначала создадим структуру для роли, которая будет устанавливать и настраивать Rundeck:

```bash
mkdir -p roles/rundeck/{tasks,handlers,templates,vars,files,defaults,meta}
cd roles/rundeck
```

## Основные файлы роли

### 1. defaults/main.yml - переменные по умолчанию

```yaml
---
# Версия Rundeck
rundeck_version: "4.14.0"

# Пользователь и группа под которыми будет работать Rundeck
rundeck_user: "rundeck"
rundeck_group: "rundeck"

# Директории
rundeck_home: "/var/lib/rundeck"
rundeck_config_dir: "/etc/rundeck"
rundeck_log_dir: "/var/log/rundeck"

# Настройки базы данных (по умолчанию встроенная H2)
rundeck_db_type: "h2"
rundeck_db_url: "jdbc:h2:file:${rundeck_home}/data/rundeckdb;MVCC=true;TRACE_LEVEL_FILE=0"
rundeck_db_driver: "org.h2.Driver"

# Настройки сервера
rundeck_server_hostname: "localhost"
rundeck_server_port: 4440
rundeck_server_address: "0.0.0.0"

# Администратор Rundeck
rundeck_admin_user: "admin"
rundeck_admin_password: "admin"  # В продакшене заменить на надежный пароль!
```

### 2. tasks/main.yml - основные задачи

```yaml
---
- name: Установка зависимостей
  package:
    name:
      - java-11-openjdk
      - curl
    state: present

- name: Создание пользователя и группы Rundeck
  user:
    name: "{{ rundeck_user }}"
    group: "{{ rundeck_group }}"
    home: "{{ rundeck_home }}"
    system: yes
    create_home: yes

- name: Создание директорий Rundeck
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ rundeck_user }}"
    group: "{{ rundeck_group }}"
    mode: '0755'
  loop:
    - "{{ rundeck_home }}"
    - "{{ rundeck_config_dir }}"
    - "{{ rundeck_log_dir }}"

- name: Загрузка Rundeck
  get_url:
    url: "https://download.rundeck.com/war/rundeck-{{ rundeck_version }}.war"
    dest: "/opt/rundeck.war"
    mode: '0644'
    owner: "{{ rundeck_user }}"
    group: "{{ rundeck_group }}"

- name: Копирование конфигурационных файлов
  template:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    owner: "{{ rundeck_user }}"
    group: "{{ rundeck_group }}"
    mode: '0640'
  with_items:
    - { src: "framework.properties.j2", dest: "{{ rundeck_config_dir }}/framework.properties" }
    - { src: "rundeck-config.properties.j2", dest: "{{ rundeck_config_dir }}/rundeck-config.properties" }

- name: Создание systemd unit файла
  template:
    src: "rundeck.service.j2"
    dest: "/etc/systemd/system/rundeck.service"
    mode: '0644'

- name: Обновление systemd
  systemd:
    daemon_reload: yes

- name: Запуск и включение службы Rundeck
  service:
    name: rundeck
    state: started
    enabled: yes

- name: Ожидание запуска Rundeck
  uri:
    url: "http://localhost:{{ rundeck_server_port }}/api/{{ rundeck_api_version }}/system/info"
    method: GET
    status_code: 200
    timeout: 30
  register: result
  until: result.status == 200
  retries: 10
  delay: 10
```

### 3. templates/rundeck.service.j2 - unit файл для systemd

```ini
[Unit]
Description=Rundeck
After=syslog.target network.target

[Service]
User={{ rundeck_user }}
Group={{ rundeck_group }}
ExecStart=/usr/bin/java -jar /opt/rundeck.war
Restart=on-failure
Environment="JAVA_HOME=/usr/lib/jvm/jre"
Environment="RDECK_BASE={{ rundeck_config_dir }}"

[Install]
WantedBy=multi-user.target
```

### 4. templates/framework.properties.j2 - конфигурация framework

```properties
framework.server.name = {{ rundeck_server_hostname }}
framework.server.hostname = {{ rundeck_server_hostname }}
framework.server.port = {{ rundeck_server_port }}
framework.server.url = http://{{ rundeck_server_hostname }}:{{ rundeck_server_port }}

framework.projects.dir = {{ rundeck_home }}/projects
framework.etc.dir = {{ rundeck_config_dir }}
framework.var.dir = {{ rundeck_home }}/var
framework.tmp.dir = {{ rundeck_home }}/var/tmp
framework.logs.dir = {{ rundeck_log_dir }}

framework.ssh.keypath = {{ rundeck_home }}/.ssh/id_rsa
framework.ssh.user = {{ rundeck_user }}

rundeck.log.level = INFO
```

### 5. templates/rundeck-config.properties.j2 - основной конфиг

```properties
grails.serverURL=http://{{ rundeck_server_hostname }}:{{ rundeck_server_port }}

# Datasource configuration
dataSource.dbCreate = update
dataSource.url = {{ rundeck_db_url }}
dataSource.driverClassName = {{ rundeck_db_driver }}
dataSource.username = 
dataSource.password = 

# Security configuration
rundeck.security.useHMacRequestTokens = true
rundeck.security.apiCookieAccess.enabled = true

# Admin user
rundeck.security.users = {{ rundeck_admin_user }}: {{ rundeck_admin_password }},user,admin,architect,deploy,build
```

## Использование роли

1. Создайте playbook `rundeck.yml`:

```yaml
---
- hosts: rundeck_servers
  become: yes
  roles:
    - rundeck
```

2. Inventory файл (`hosts` или другой):

```ini
[rundeck_servers]
rundeck.example.com
```

3. Запуск playbook:

```bash
ansible-playbook -i hosts rundeck.yml
```

## Дополнительные улучшения

1. **Настройка внешней БД** (MySQL/PostgreSQL):

Добавьте в `vars/main.yml`:

```yaml
rundeck_db_type: "mysql"
rundeck_db_url: "jdbc:mysql://mysql-host:3306/rundeck?autoReconnect=true&useSSL=false"
rundeck_db_driver: "com.mysql.jdbc.Driver"
rundeck_db_username: "rundeck"
rundeck_db_password: "securepassword"
```

И добавьте задачу для установки JDBC драйвера:

```yaml
- name: Установка MySQL JDBC драйвера
  get_url:
    url: "https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-8.0.23.tar.gz"
    dest: "/tmp/mysql-connector.tar.gz"
  when: rundeck_db_type == "mysql"

- name: Распаковка MySQL JDBC драйвера
  unarchive:
    src: "/tmp/mysql-connector.tar.gz"
    dest: "/tmp/"
    remote_src: yes
  when: rundeck_db_type == "mysql"

- name: Копирование MySQL JDBC драйвера
  copy:
    src: "/tmp/mysql-connector-java-8.0.23/mysql-connector-java-8.0.23.jar"
    dest: "/var/lib/rundeck/bootstrap/"
    owner: "{{ rundeck_user }}"
    group: "{{ rundeck_group }}"
    mode: '0644'
  when: rundeck_db_type == "mysql"
```

2. **Настройка HTTPS**:

Добавьте задачи для настройки reverse proxy (Nginx/Apache) и получения SSL сертификатов (Let's Encrypt).

3. **Настройка кластера Rundeck**:

Для кластерной установки потребуется дополнительная настройка shared storage для проектов и конфигурации.