# Ansible роль для установки агента BI.ZONE EDR в Linux

Вот пример Ansible роли для установки агента BI.ZONE EDR в Linux системах (RPM и DEB):

## Структура роли

```
roles/install_bz_agent/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   └── env_vars.j2
└── vars
    └── main.yml
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки по умолчанию
bz_agent_package_path: ""
bz_agent_type: ""  # rpm или deb

# Параметры агента
bz_authority_service: ""
bz_sensors_service: ""
bz_polling_period: "300s"
bz_dial_timeout: "15s"
bz_agent_groups: ""
```

### tasks/main.yml

```yaml
---
- name: Проверка обязательных параметров
  assert:
    that:
      - bz_agent_package_path != ""
      - bz_agent_type in ['rpm', 'deb']
      - bz_authority_service != ""
      - bz_sensors_service != ""
    fail_msg: "Не указаны обязательные параметры для установки агента"

- name: Создание временного файла с переменными окружения
  template:
    src: env_vars.j2
    dest: /tmp/bz_agent_env_vars
    mode: 0644

- name: Установка RPM пакета
  when: bz_agent_type == 'rpm'
  ansible.builtin.command: |
    set -a && . /tmp/bz_agent_env_vars && sudo -E yum install -y {{ bz_agent_package_path }}
  args:
    executable: /bin/bash

- name: Установка DEB пакета
  when: bz_agent_type == 'deb'
  ansible.builtin.command: |
    set -a && . /tmp/bz_agent_env_vars && sudo -E apt-get install -y {{ bz_agent_package_path }}
  args:
    executable: /bin/bash

- name: Проверка состояния службы bzsenagent
  ansible.builtin.command: systemctl status bzsenagent.service
  register: service_status
  ignore_errors: yes

- name: Вывод статуса службы
  ansible.builtin.debug:
    var: service_status.stdout_lines

- name: Удаление временного файла с переменными
  ansible.builtin.file:
    path: /tmp/bz_agent_env_vars
    state: absent
```

### templates/env_vars.j2

```bash
# Environment variables for BI.ZONE EDR agent installation
export BZ_AUTHORITY_SERVICE={{ bz_authority_service }}
export BZ_SENSORS_SERVICE={{ bz_sensors_service }}
export BZ_POLLING_PERIOD={{ bz_polling_period }}
export BZ_DIAL_TIMEOUT={{ bz_dial_timeout }}
{% if bz_agent_groups != "" %}
export BZ_AGENT_GROUPS='{{ bz_agent_groups }}'
{% endif %}
```

## Пример использования роли

```yaml
- hosts: linux_servers
  become: yes
  roles:
    - role: install_bz_agent
      vars:
        bz_agent_package_path: "/path/to/bzagent.rpm"
        bz_agent_type: "rpm"
        bz_authority_service: "management.example.com:80"
        bz_sensors_service: "management.example.com:443"
        bz_polling_period: "300s"
        bz_dial_timeout: "15s"
        bz_agent_groups: "nixprod"
```

## Примечания

1. Роль поддерживает установку как RPM, так и DEB пакетов
2. Все переменные окружения передаются с префиксом BZ_ как требуется
3. Используется sudo -E для сохранения переменных окружения
4. После установки проверяется статус службы bzsenagent.service
5. Для работы роли необходимо указать обязательные параметры:
   - Путь к пакету
   - Тип пакета (rpm/deb)
   - Адреса серверов управления (authority и sensors)

Роль можно расширить для поддержки дополнительных параметров агента или добавить обработку ошибок.

# Универсальная Ansible роль для установки агента BI.ZONE EDR

Вот улучшенная универсальная роль, которая поддерживает различные сценарии установки, проверки и обновления агента.

## Структура роли

```
roles/bzone_edr_agent/
├── defaults/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── install_deb.yml
│   ├── install_rpm.yml
│   ├── configure.yml
│   ├── service.yml
│   └── verify.yml
├── templates/
│   └── bz-agent.conf.j2
├── vars/
│   └── main.yml
└── handlers/
    └── main.yml
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Основные параметры
bz_agent_install: true
bz_agent_upgrade: false
bz_agent_force_install: false

# Тип пакета (auto/rpm/deb)
bz_agent_pkg_type: "auto"

# Путь к пакету (если не указан, будет попытка установки из репозитория)
bz_agent_pkg_path: ""

# Параметры подключения (обязательные)
bz_authority_service: ""
bz_sensors_service: ""

# Дополнительные параметры
bz_polling_period: "300s"
bz_dial_timeout: "15s"
bz_agent_groups: ""
bz_agent_tags: ""
bz_agent_http_proxy: ""
bz_agent_https_proxy: ""
bz_agent_no_proxy: ""

# Настройки сервиса
bz_agent_service_state: "started"
bz_agent_service_enabled: true

# Проверка установки
bz_agent_verify_installation: true
bz_agent_verify_timeout: 120
```

### tasks/main.yml

```yaml
---
- name: Проверка предварительных условий
  ansible.builtin.assert:
    that:
      - bz_authority_service != ""
      - bz_sensors_service != ""
    msg: "Не указаны обязательные параметры bz_authority_service и bz_sensors_service"

- name: Определение типа пакета (если auto)
  ansible.builtin.set_fact:
    bz_agent_actual_pkg_type: "{{ 'deb' if ansible_facts['pkg_mgr'] == 'apt' else 'rpm' }}"
  when: bz_agent_pkg_type == "auto"

- name: Установка фактического типа пакета
  ansible.builtin.set_fact:
    bz_agent_actual_pkg_type: "{{ bz_agent_pkg_type }}"
  when: bz_agent_pkg_type != "auto"

- name: Проверка установленного агента
  ansible.builtin.command: which bzsenagent
  register: bz_agent_installed
  ignore_errors: yes
  changed_when: false

- name: Получение текущей версии агента
  ansible.builtin.command: bzsenagent --version
  register: bz_agent_version
  changed_when: false
  ignore_errors: yes

- name: Установка факта наличия агента
  ansible.builtin.set_fact:
    bz_agent_is_installed: "{{ bz_agent_installed.rc == 0 }}"
    bz_agent_current_version: "{{ bz_agent_version.stdout | default('') }}"

- name: Пропуск установки (если агент уже установлен и не требуется принудительная установка)
  ansible.builtin.meta: end_play
  when: bz_agent_is_installed and not bz_agent_force_install and not bz_agent_upgrade

- include_tasks: install_{{ bz_agent_actual_pkg_type }}.yml
  when: bz_agent_install

- include_tasks: configure.yml
  when: bz_agent_install or bz_agent_upgrade

- include_tasks: service.yml

- include_tasks: verify.yml
  when: bz_agent_verify_installation
```

### tasks/install_deb.yml

```yaml
---
- name: Установка зависимостей для DEB
  ansible.builtin.apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
    state: present
    update_cache: yes

- name: Добавление GPG ключа репозитория BI.ZONE
  ansible.builtin.apt_key:
    url: https://repo.bi.zone/keys/bz-apt-key.gpg
    state: present
  when: bz_agent_pkg_path == ""

- name: Добавление репозитория BI.ZONE
  ansible.builtin.apt_repository:
    repo: "deb [arch=amd64] https://repo.bi.zone/apt stable main"
    state: present
    update_cache: yes
  when: bz_agent_pkg_path == ""

- name: Установка агента из DEB пакета
  ansible.builtin.apt:
    deb: "{{ bz_agent_pkg_path }}"
    state: present
  environment: "{{ bz_agent_env_vars }}"
  when: bz_agent_pkg_path != ""

- name: Установка агента из репозитория
  ansible.builtin.apt:
    name: bzsenagent
    state: "{{ 'latest' if bz_agent_upgrade else 'present' }}"
    update_cache: yes
  environment: "{{ bz_agent_env_vars }}"
  when: bz_agent_pkg_path == ""
```

### tasks/install_rpm.yml

```yaml
---
- name: Установка зависимостей для RPM
  ansible.builtin.yum:
    name:
      - yum-utils
      - curl
    state: present

- name: Добавление репозитория BI.ZONE
  ansible.builtin.get_url:
    url: https://repo.bi.zone/yum/bz.repo
    dest: /etc/yum.repos.d/bz.repo
    mode: '0644'
  when: bz_agent_pkg_path == ""

- name: Установка агента из RPM пакета
  ansible.builtin.yum:
    name: "{{ bz_agent_pkg_path }}"
    state: present
    disable_gpg_check: yes
  environment: "{{ bz_agent_env_vars }}"
  when: bz_agent_pkg_path != ""

- name: Установка агента из репозитория
  ansible.builtin.yum:
    name: bzsenagent
    state: "{{ 'latest' if bz_agent_upgrade else 'present' }}"
  environment: "{{ bz_agent_env_vars }}"
  when: bz_agent_pkg_path == ""
```

### tasks/configure.yml

```yaml
---
- name: Создание конфигурационного файла агента
  ansible.builtin.template:
    src: bz-agent.conf.j2
    dest: /etc/bzsenagent/config.yaml
    owner: root
    group: root
    mode: '0640'
  notify: Restart bzsenagent service
```

### tasks/service.yml

```yaml
---
- name: Управление сервисом bzsenagent
  ansible.builtin.systemd:
    name: bzsenagent
    state: "{{ bz_agent_service_state }}"
    enabled: "{{ bz_agent_service_enabled }}"
    daemon_reload: yes
```

### tasks/verify.yml

```yaml
---
- name: Ожидание запуска агента
  ansible.builtin.wait_for:
    path: /var/run/bzsenagent/bzsenagent.pid
    state: present
    timeout: "{{ bz_agent_verify_timeout }}"

- name: Проверка статуса агента
  ansible.builtin.command: bzsenagent status
  register: agent_status
  changed_when: false

- name: Вывод статуса агента
  ansible.builtin.debug:
    var: agent_status.stdout_lines
```

### templates/bz-agent.conf.j2

```yaml
# BI.ZONE EDR Agent Configuration
# Managed by Ansible - DO NOT EDIT MANUALLY

authority_service: "{{ bz_authority_service }}"
sensors_service: "{{ bz_sensors_service }}"
polling_period: "{{ bz_polling_period }}"
dial_timeout: "{{ bz_dial_timeout }}"

{% if bz_agent_groups %}
agent_groups: [{{ bz_agent_groups.split(',') | map('trim') | join(', ') }}]
{% endif %}

{% if bz_agent_tags %}
tags:
  {% for tag in bz_agent_tags.split(',') %}
  - {{ tag | trim }}
  {% endfor %}
{% endif %}

{% if bz_agent_http_proxy %}
http_proxy: "{{ bz_agent_http_proxy }}"
{% endif %}

{% if bz_agent_https_proxy %}
https_proxy: "{{ bz_agent_https_proxy }}"
{% endif %}

{% if bz_agent_no_proxy %}
no_proxy: "{{ bz_agent_no_proxy }}"
{% endif %}
```

### handlers/main.yml

```yaml
---
- name: Restart bzsenagent service
  ansible.builtin.systemd:
    name: bzsenagent
    state: restarted
    enabled: yes
    daemon_reload: yes
```

## Пример использования роли

### Установка из репозитория

```yaml
- hosts: all
  roles:
    - role: bzone_edr_agent
      vars:
        bz_authority_service: "mgmt.example.com:80"
        bz_sensors_service: "mgmt.example.com:443"
        bz_agent_groups: "prod,linux"
        bz_agent_tags: "env=production,role=webserver"
```

### Установка из локального пакета

```yaml
- hosts: all
  roles:
    - role: bzone_edr_agent
      vars:
        bz_agent_pkg_path: "/tmp/bzsenagent-latest.rpm"
        bz_authority_service: "mgmt.example.com:80"
        bz_sensors_service: "mgmt.example.com:443"
        bz_agent_force_install: true
```

### Обновление агента

```yaml
- hosts: all
  roles:
    - role: bzone_edr_agent
      vars:
        bz_agent_upgrade: true
        bz_authority_service: "mgmt.example.com:80"
        bz_sensors_service: "mgmt.example.com:443"
```

## Особенности роли

1. **Автоматическое определение типа ОС** (RPM/DEB) при `bz_agent_pkg_type: "auto"`
2. **Поддержка нескольких сценариев**:
   - Установка из репозитория BI.ZONE
   - Установка из локального пакета
   - Обновление существующего агента
   - Принудительная переустановка
3. **Гибкая конфигурация**:
   - Поддержка групп и тегов агента
   - Настройка прокси
   - Контроль параметров подключения
4. **Проверка установки**:
   - Верификация установки
   - Проверка версии
   - Контроль состояния сервиса
5. **Идемпотентность** - роль можно запускать многократно без побочных эффектов

Роль может быть легко расширена для поддержки дополнительных параметров конфигурации или интеграции с другими системами.

# Универсальная Ansible роль для установки агента BI.ZONE EDR с локальными пакетами

Вот модифицированная версия роли, где RPM и DEB пакеты хранятся в директории `files` самой роли.

## Обновленная структура роли

```
roles/bzone_edr_agent/
├── files/
│   ├── bzsenagent-latest.rpm
│   └── bzsenagent-latest.deb
├── defaults/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── install_deb.yml
│   ├── install_rpm.yml
│   ├── configure.yml
│   ├── service.yml
│   └── verify.yml
├── templates/
│   └── bz-agent.conf.j2
├── vars/
│   └── main.yml
└── handlers/
    └── main.yml
```

## Ключевые изменения

### defaults/main.yml

```yaml
---
# Основные параметры
bz_agent_install: true
bz_agent_upgrade: false
bz_agent_force_install: false

# Тип пакета (auto/rpm/deb)
bz_agent_pkg_type: "auto"

# Использование локальных пакетов из роли
bz_agent_use_local_pkg: true

# Имена локальных пакетов (хранятся в files/)
bz_agent_local_rpm: "bzsenagent-latest.rpm"
bz_agent_local_deb: "bzsenagent-latest.deb"

# Параметры подключения (обязательные)
bz_authority_service: ""
bz_sensors_service: ""
```

### tasks/main.yml (добавлен блок)

```yaml
- name: Определение пути к локальному пакету
  ansible.builtin.set_fact:
    bz_agent_pkg_path: "{{ role_path }}/files/{{ bz_agent_local_rpm if bz_agent_actual_pkg_type == 'rpm' else bz_agent_local_deb }}"
  when: bz_agent_use_local_pkg
```

### tasks/install_deb.yml (обновлено)

```yaml
- name: Копирование DEB пакета на целевой хост
  ansible.builtin.copy:
    src: "{{ bz_agent_pkg_path }}"
    dest: "/tmp/{{ bz_agent_local_deb }}"
    remote_src: false
  when: bz_agent_use_local_pkg

- name: Установка агента из локального DEB пакета
  ansible.builtin.apt:
    deb: "/tmp/{{ bz_agent_local_deb }}"
    state: present
  environment: "{{ bz_agent_env_vars }}"
  when: bz_agent_use_local_pkg
```

### tasks/install_rpm.yml (обновлено)

```yaml
- name: Копирование RPM пакета на целевой хост
  ansible.builtin.copy:
    src: "{{ bz_agent_pkg_path }}"
    dest: "/tmp/{{ bz_agent_local_rpm }}"
    remote_src: false
  when: bz_agent_use_local_pkg

- name: Установка агента из локального RPM пакета
  ansible.builtin.yum:
    name: "/tmp/{{ bz_agent_local_rpm }}"
    state: present
    disable_gpg_check: yes
  environment: "{{ bz_agent_env_vars }}"
  when: bz_agent_use_local_pkg
```

## Полный обновленный код роли

### defaults/main.yml

```yaml
---
# Основные параметры
bz_agent_install: true
bz_agent_upgrade: false
bz_agent_force_install: false

# Тип пакета (auto/rpm/deb)
bz_agent_pkg_type: "auto"

# Использование локальных пакетов из роли
bz_agent_use_local_pkg: true

# Имена локальных пакетов (хранятся в files/)
bz_agent_local_rpm: "bzsenagent-latest.rpm"
bz_agent_local_deb: "bzsenagent-latest.deb"

# Параметры подключения (обязательные)
bz_authority_service: ""
bz_sensors_service: ""

# Дополнительные параметры
bz_polling_period: "300s"
bz_dial_timeout: "15s"
bz_agent_groups: ""
bz_agent_tags: ""
bz_agent_http_proxy: ""
bz_agent_https_proxy: ""
bz_agent_no_proxy: ""

# Настройки сервиса
bz_agent_service_state: "started"
bz_agent_service_enabled: true

# Проверка установки
bz_agent_verify_installation: true
bz_agent_verify_timeout: 120
```

### tasks/main.yml

```yaml
---
- name: Проверка предварительных условий
  ansible.builtin.assert:
    that:
      - bz_authority_service != ""
      - bz_sensors_service != ""
    msg: "Не указаны обязательные параметры bz_authority_service и bz_sensors_service"

- name: Определение типа пакета (если auto)
  ansible.builtin.set_fact:
    bz_agent_actual_pkg_type: "{{ 'deb' if ansible_facts['pkg_mgr'] == 'apt' else 'rpm' }}"
  when: bz_agent_pkg_type == "auto"

- name: Установка фактического типа пакета
  ansible.builtin.set_fact:
    bz_agent_actual_pkg_type: "{{ bz_agent_pkg_type }}"
  when: bz_agent_pkg_type != "auto"

- name: Определение пути к локальному пакету
  ansible.builtin.set_fact:
    bz_agent_pkg_path: "{{ role_path }}/files/{{ bz_agent_local_rpm if bz_agent_actual_pkg_type == 'rpm' else bz_agent_local_deb }}"
  when: bz_agent_use_local_pkg

- name: Проверка установленного агента
  ansible.builtin.command: which bzsenagent
  register: bz_agent_installed
  ignore_errors: yes
  changed_when: false

- name: Получение текущей версии агента
  ansible.builtin.command: bzsenagent --version
  register: bz_agent_version
  changed_when: false
  ignore_errors: yes

- name: Установка факта наличия агента
  ansible.builtin.set_fact:
    bz_agent_is_installed: "{{ bz_agent_installed.rc == 0 }}"
    bz_agent_current_version: "{{ bz_agent_version.stdout | default('') }}"

- name: Пропуск установки (если агент уже установлен и не требуется принудительная установка)
  ansible.builtin.meta: end_play
  when: bz_agent_is_installed and not bz_agent_force_install and not bz_agent_upgrade

- include_tasks: install_{{ bz_agent_actual_pkg_type }}.yml
  when: bz_agent_install

- include_tasks: configure.yml
  when: bz_agent_install or bz_agent_upgrade

- include_tasks: service.yml

- include_tasks: verify.yml
  when: bz_agent_verify_installation
```

### tasks/install_deb.yml

```yaml
---
- name: Установка зависимостей для DEB
  ansible.builtin.apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
    state: present
    update_cache: yes
  when: not bz_agent_use_local_pkg

- name: Добавление GPG ключа репозитория BI.ZONE
  ansible.builtin.apt_key:
    url: https://repo.bi.zone/keys/bz-apt-key.gpg
    state: present
  when: not bz_agent_use_local_pkg

- name: Добавление репозитория BI.ZONE
  ansible.builtin.apt_repository:
    repo: "deb [arch=amd64] https://repo.bi.zone/apt stable main"
    state: present
    update_cache: yes
  when: not bz_agent_use_local_pkg

- name: Копирование DEB пакета на целевой хост
  ansible.builtin.copy:
    src: "{{ bz_agent_pkg_path }}"
    dest: "/tmp/{{ bz_agent_local_deb }}"
    remote_src: false
  when: bz_agent_use_local_pkg

- name: Установка агента из локального DEB пакета
  ansible.builtin.apt:
    deb: "/tmp/{{ bz_agent_local_deb }}"
    state: present
  environment: "{{ bz_agent_env_vars }}"
  when: bz_agent_use_local_pkg

- name: Установка агента из репозитория
  ansible.builtin.apt:
    name: bzsenagent
    state: "{{ 'latest' if bz_agent_upgrade else 'present' }}"
    update_cache: yes
  environment: "{{ bz_agent_env_vars }}"
  when: not bz_agent_use_local_pkg

- name: Удаление временного DEB пакета
  ansible.builtin.file:
    path: "/tmp/{{ bz_agent_local_deb }}"
    state: absent
  when: bz_agent_use_local_pkg
```

### tasks/install_rpm.yml

```yaml
---
- name: Установка зависимостей для RPM
  ansible.builtin.yum:
    name:
      - yum-utils
      - curl
    state: present
  when: not bz_agent_use_local_pkg

- name: Добавление репозитория BI.ZONE
  ansible.builtin.get_url:
    url: https://repo.bi.zone/yum/bz.repo
    dest: /etc/yum.repos.d/bz.repo
    mode: '0644'
  when: not bz_agent_use_local_pkg

- name: Копирование RPM пакета на целевой хост
  ansible.builtin.copy:
    src: "{{ bz_agent_pkg_path }}"
    dest: "/tmp/{{ bz_agent_local_rpm }}"
    remote_src: false
  when: bz_agent_use_local_pkg

- name: Установка агента из локального RPM пакета
  ansible.builtin.yum:
    name: "/tmp/{{ bz_agent_local_rpm }}"
    state: present
    disable_gpg_check: yes
  environment: "{{ bz_agent_env_vars }}"
  when: bz_agent_use_local_pkg

- name: Установка агента из репозитория
  ansible.builtin.yum:
    name: bzsenagent
    state: "{{ 'latest' if bz_agent_upgrade else 'present' }}"
  environment: "{{ bz_agent_env_vars }}"
  when: not bz_agent_use_local_pkg

- name: Удаление временного RPM пакета
  ansible.builtin.file:
    path: "/tmp/{{ bz_agent_local_rpm }}"
    state: absent
  when: bz_agent_use_local_pkg
```

## Пример использования роли

```yaml
- hosts: all
  roles:
    - role: bzone_edr_agent
      vars:
        bz_authority_service: "mgmt.example.com:80"
        bz_sensors_service: "mgmt.example.com:443"
        bz_agent_groups: "prod,linux"
        bz_agent_tags: "env=production,role=webserver"
        bz_agent_use_local_pkg: true
```

## Особенности обновленной роли

1. **Локальное хранение пакетов** - RPM и DEB пакеты хранятся в директории `files` роли
2. **Гибкое управление источниками установки**:
   - Установка из локальных пакетов (`bz_agent_use_local_pkg: true`)
   - Установка из официального репозитория (`bz_agent_use_local_pkg: false`)
3. **Автоматическая очистка** - временные пакеты удаляются после установки
4. **Совместимость с предыдущими версиями** - все оригинальные функции сохранены
5. **Улучшенная обработка зависимостей** - установка только при необходимости

Роль теперь полностью самодостаточна и может работать в изолированных средах без доступа к интернету, при условии размещения пакетов в директории `files`.




Чтобы повысить права (privilege escalation) при установке DEB-пакета через модуль `apt`, нужно добавить директиву `become: yes`. Вот исправленная версия задачи:

```yaml
- name: Установка агента из локального DEB пакета
  become: yes                # Повышение прав до root
  become_method: sudo        # Метод повышения прав (по умолчанию sudo)
  ansible.builtin.apt:
    deb: "/tmp/{{ bz_agent_local_deb }}"
    state: present
  environment: "{{ bz_agent_env_vars }}"
  when: bz_agent_use_local_pkg
```

Ключевые изменения:
1. Добавлена директива `become: yes` - включает повышение привилегий
2. Добавлена `become_method: sudo` (можно опустить, так как это значение по умолчанию)

Полная улучшенная версия задачи с обработкой ошибок:

```yaml
- name: Установка агента из локального DEB пакета
  become: yes
  ansible.builtin.apt:
    deb: "/tmp/{{ bz_agent_local_deb }}"
    state: present
    allow_unauthenticated: yes  # Если пакет не подписан
    install_recommends: no      # Не устанавливать рекомендованные пакеты
  environment: "{{ bz_agent_env_vars }}"
  register: deb_install
  ignore_errors: yes
  when: bz_agent_use_local_pkg

- name: Проверка успешности установки
  ansible.builtin.assert:
    that:
      - deb_install is success or "is already the newest version" in deb_install.stderr
    fail_msg: "Не удалось установить DEB пакет"
    success_msg: "Пакет успешно установлен"
```

Дополнительные рекомендации:

1. На уровне playbook можно задать глобальные настройки повышения прав:
```yaml
- hosts: all
  become: yes               # Повышать права для всех задач
  become_method: sudo       # Использовать sudo
  become_user: root         # Переключаться на root (по умолчанию)
  roles:
    - role: bzone_edr_agent
```

2. Убедитесь, что пользователь Ansible имеет права sudo без пароля:
```bash
# В файле /etc/sudoers на целевых хостах
ansible_user ALL=(ALL) NOPASSWD: ALL
```

3. Для сложных случаев можно использовать `command` вместо `apt`:
```yaml
- name: Альтернативная установка через dpkg
  become: yes
  ansible.builtin.command: >
    dpkg -i --force-confdef --force-confold "/tmp/{{ bz_agent_local_deb }}"
  args:
    creates: "/usr/bin/bzsenagent"  # Проверка существования файла для идемпотентности
```