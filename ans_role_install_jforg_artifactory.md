# Роль Ansible для установки и настройки JFrog Artifactory

Ниже представлена базовая структура роли Ansible для установки и настройки JFrog Artifactory.

## Структура роли

```
roles/artifactory/
├── defaults
│   └── main.yml       # Значения переменных по умолчанию
├── files
│   └── systemd         # Файлы конфигурации systemd (если нужны)
├── handlers
│   └── main.yml       # Обработчики (например, перезапуск сервиса)
├── tasks
│   ├── install.yml    # Установка Artifactory
│   ├── configure.yml  # Настройка Artifactory
│   ├── service.yml    # Управление сервисом
│   └── main.yml       # Основной файл задач
├── templates
│   └── systemd.j2     # Шаблоны конфигурации (если нужны)
└── vars
    └── main.yml       # Переменные роли
```

## Основные файлы роли

### defaults/main.yml

```yaml
---
# Версия Artifactory
artifactory_version: "7.41.13"
artifactory_download_url: "https://releases.jfrog.io/artifactory/artifactory-pro/org/artifactory/pro/jfrog-artifactory-pro/{{ artifactory_version }}/jfrog-artifactory-pro-{{ artifactory_version }}.zip"

# Директории установки
artifactory_home: "/var/opt/jfrog/artifactory"
artifactory_data: "{{ artifactory_home }}/data"
artifactory_backup: "{{ artifactory_home }}/backup"

# Пользователь и группа
artifactory_user: "artifactory"
artifactory_group: "artifactory"

# Настройки сервиса
artifactory_service_name: "artifactory"
artifactory_service_state: "started"
artifactory_service_enabled: true

# Настройки Java
java_home: "/usr/lib/jvm/java-11-openjdk"
java_opts: "-Xms512m -Xmx4g -Xss256k -XX:+UseG1GC"
```

### tasks/main.yml

```yaml
---
- name: Include install tasks
  include_tasks: install.yml

- name: Include configure tasks
  include_tasks: configure.yml

- name: Include service tasks
  include_tasks: service.yml
```

### tasks/install.yml

```yaml
---
- name: Ensure Java is installed
  package:
    name: "openjdk-11-jdk"
    state: present

- name: Create Artifactory user and group
  user:
    name: "{{ artifactory_user }}"
    group: "{{ artifactory_group }}"
    system: yes
    create_home: no
    shell: /bin/false

- name: Create Artifactory directories
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ artifactory_user }}"
    group: "{{ artifactory_group }}"
    mode: '0755'
  loop:
    - "{{ artifactory_home }}"
    - "{{ artifactory_data }}"
    - "{{ artifactory_backup }}"

- name: Download Artifactory
  get_url:
    url: "{{ artifactory_download_url }}"
    dest: "/tmp/artifactory.zip"
    mode: '0755'
    timeout: 60

- name: Unpack Artifactory
  unarchive:
    src: "/tmp/artifactory.zip"
    dest: "{{ artifactory_home }}"
    remote_src: yes
    extra_opts: "--strip-components=1"
    owner: "{{ artifactory_user }}"
    group: "{{ artifactory_group }}"

- name: Clean up download
  file:
    path: "/tmp/artifactory.zip"
    state: absent
```

### tasks/configure.yml

```yaml
---
- name: Configure system.properties
  template:
    src: "templates/system.properties.j2"
    dest: "{{ artifactory_home }}/etc/system.properties"
    owner: "{{ artifactory_user }}"
    group: "{{ artifactory_group }}"
    mode: '0644'

- name: Configure artifactory.config.import.xml
  template:
    src: "templates/artifactory.config.import.xml.j2"
    dest: "{{ artifactory_home }}/etc/artifactory.config.import.xml"
    owner: "{{ artifactory_user }}"
    group: "{{ artifactory_group }}"
    mode: '0644'

- name: Set Java environment variables
  lineinfile:
    path: "{{ artifactory_home }}/bin/artifactory.default"
    regexp: "^export JAVA_HOME="
    line: "export JAVA_HOME={{ java_home }}"
    state: present

- name: Set Java options
  lineinfile:
    path: "{{ artifactory_home }}/bin/artifactory.default"
    regexp: "^export JAVA_OPTIONS="
    line: "export JAVA_OPTIONS={{ java_opts }}"
    state: present
```

### tasks/service.yml

```yaml
---
- name: Install Artifactory systemd service
  template:
    src: "templates/artifactory.service.j2"
    dest: "/etc/systemd/system/{{ artifactory_service_name }}.service"
    mode: '0644'

- name: Reload systemd daemon
  systemd:
    daemon_reload: yes

- name: Ensure Artifactory service is {{ artifactory_service_state }} and enabled
  systemd:
    name: "{{ artifactory_service_name }}"
    state: "{{ artifactory_service_state }}"
    enabled: "{{ artifactory_service_enabled }}"
```

### handlers/main.yml

```yaml
---
- name: restart artifactory
  systemd:
    name: "{{ artifactory_service_name }}"
    state: restarted
    daemon_reload: yes
```

## Пример playbook

```yaml
---
- name: Install and configure JFrog Artifactory
  hosts: artifactory_servers
  become: yes
  roles:
    - artifactory
```

## Дополнительные настройки

Для полной настройки Artifactory вам может потребоваться:

1. Настройка обратного прокси (Nginx/Apache)
2. Настройка базы данных (PostgreSQL/MySQL/Oracle)
3. Настройка хранилища (S3, Google Storage и т.д.)
4. Настройка репликации для HA
5. Настройка SSL сертификатов

Вы можете расширить роль, добавив соответствующие задачи и шаблоны конфигурации.