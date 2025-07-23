# Универсальная Ansible роль для загрузки публичных ключей репозиториев

Вот улучшенная универсальная версия роли, которая работает для любых дистрибутивов и позволяет гибко настраивать ключи через переменные.

## Улучшения:
1. Поддержка любых дистрибутивов без жесткой привязки к версиям
2. Гибкая настройка через переменные
3. Возможность переопределения URL ключей
4. Автоматическое создание структуры каталогов
5. Проверка целостности ключей

## Структура роли

```
roles/repo_keys/
├── tasks/
│   └── main.yml
├── defaults/
│   └── main.yml
├── vars/
│   └── main.yml
└── templates/
    └── repo-keys.conf.j2
```

## Содержание файлов

### defaults/main.yml (значения по умолчанию)

```yaml
---
# Базовый каталог для хранения ключей
repo_keys_base_dir: "/etc/apt/keys"  # Для Debian/Ubuntu
# repo_keys_base_dir: "/etc/pki/rpm-gpg"  # Для RHEL/CentOS/OracleLinux

# Формат подкаталогов: by_distro, by_repo или flat
repo_keys_dir_structure: "by_distro"

# Список ключей для загрузки (можно переопределить)
repo_keys_to_download: []
  # Пример структуры:
  # - name: "Oracle Linux"
  #   url: "https://yum.oracle.com/RPM-GPG-KEY-oracle-ol9"
  #   dest: "RPM-GPG-KEY-oracle"  # опционально
  #   distro: "OracleLinux"       # опционально
  #   versions: ["7", "8", "9"]   # опционально
  #   checksum: "sha256:..."      # опционально

# Политика обновления ключей
repo_keys_update_policy: "if_changed"  # always, if_changed, never
```

### vars/main.yml (основные переменные)

```yaml
---
# Общие ключи для всех дистрибутивов
common_repo_keys:
  - name: "Microsoft GPG"
    url: "https://packages.microsoft.com/keys/microsoft.asc"
  
  - name: "Docker CE"
    url: "https://download.docker.com/linux/{{ ansible_distribution | lower }}/gpg"

# Дистрибутив-специфичные ключи
distro_specific_keys:
  OracleLinux:
    - name: "Oracle Linux"
      url: "https://yum.oracle.com/RPM-GPG-KEY-oracle-ol{{ ansible_distribution_major_version }}"
  
  Ubuntu:
    - name: "Ubuntu Archive"
      url: "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x{{ '871920D1991BC93C' if ansible_distribution_major_version == '24' else '3B4FE6ACC0B21F32' }}"
  
  Debian:
    - name: "Debian Archive"
      url: "https://ftp-master.debian.org/keys/archive-key-{{ ansible_distribution_major_version }}.asc"

# Специфичные ключи для репозиториев
repo_specific_keys:
  - name: "Elasticsearch"
    url: "https://artifacts.elastic.co/GPG-KEY-elasticsearch"
    repos: ["elasticsearch"]
  
  - name: "PostgreSQL"
    url: "https://www.postgresql.org/media/keys/ACCC4CF8.asc"
    repos: ["postgresql"]
```

### tasks/main.yml

```yaml
---
- name: "Определяем базовый каталог для ключей"
  ansible.builtin.set_fact:
    actual_keys_dir: >-
      {% if ansible_os_family == 'Debian' %}/etc/apt/keys
      {% elif ansible_os_family == 'RedHat' %}/etc/pki/rpm-gpg
      {% else %}{{ repo_keys_base_dir }}{% endif %}

- name: "Определяем конечную структуру каталогов"
  ansible.builtin.set_fact:
    keys_destination: >-
      {% if repo_keys_dir_structure == 'by_distro' %}{{ actual_keys_dir }}/{{ ansible_distribution | lower }}{{ ansible_distribution_major_version }}
      {% elif repo_keys_dir_structure == 'by_repo' %}{{ actual_keys_dir }}/{% if item.repos is defined %}{{ item.repos | first }}{% else %}misc{% endif %}
      {% else %}{{ actual_keys_dir }}{% endif %}

- name: "Создаем необходимые каталоги"
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - "{{ actual_keys_dir }}"
    - "{{ actual_keys_dir }}/backups"
  loop_control:
    label: "{{ item }}"

- name: "Собираем все ключи для загрузки"
  ansible.builtin.set_fact:
    all_keys_to_download: >-
      {{
        (repo_keys_to_download if repo_keys_to_download is defined else []) +
        (common_repo_keys if common_repo_keys is defined else []) +
        (distro_specific_keys[ansible_distribution] | default([])) +
        (repo_specific_keys if repo_specific_keys is defined else [])
      }}

- name: "Фильтруем ключи по дистрибутиву и версии"
  ansible.builtin.set_fact:
    filtered_keys: >-
      {{
        all_keys_to_download | selectattr('distro', 'undefined') | list +
        (all_keys_to_download | selectattr('distro', 'defined') | selectattr('distro', 'match', ansible_distribution) | list) |
        selectattr('versions', 'undefined') | list +
        (all_keys_to_download | selectattr('versions', 'defined') | selectattr('versions', 'contains', ansible_distribution_major_version) | list)
      }}

- name: "Скачиваем ключи репозиториев"
  ansible.builtin.get_url:
    url: "{{ item.url }}"
    dest: >-
      {{ keys_destination }}/{{ 
        item.dest | default(item.url | regex_replace('^.*/([^/]+)$', '\\1') | regex_replace('[?&].*$', '')) 
      }}
    mode: '0644'
    force: "{{ repo_keys_update_policy == 'always' }}"
    checksum: "{{ item.checksum | default(omit) }}"
    backup: yes
  loop: "{{ filtered_keys }}"
  loop_control:
    label: "{{ item.name }}"
  register: keys_download
  retries: 3
  delay: 5
  until: keys_download is succeeded

- name: "Проверяем валидность ключей (для GPG)"
  ansible.builtin.command: >
    gpg --dry-run --import {{ item.dest }}
  loop: "{{ keys_download.results }}"
  when: 
    - item.dest is defined
    - ansible_os_family == 'Debian' or ansible_os_family == 'RedHat'
  ignore_errors: yes
  register: gpg_check
  changed_when: false

- name: "Отмечаем невалидные ключи"
  ansible.builtin.set_fact:
    invalid_keys: >-
      {{ gpg_check.results | selectattr('rc', '!=', 0) | 
         map(attribute='item.dest') | list }}

- name: "Уведомляем о невалидных ключах"
  ansible.builtin.debug:
    msg: "Обнаружены невалидные ключи: {{ invalid_keys }}"
  when: invalid_keys | length > 0

- name: "Сохраняем информацию о ключах"
  ansible.builtin.template:
    src: "repo-keys.conf.j2"
    dest: "{{ actual_keys_dir }}/repo-keys.info"
    mode: '0644'
```

### templates/repo-keys.conf.j2

```
# Информация о ключах репозиториев
# Сгенерировано Ansible {{ ansible_date_time.iso8601 }}

Дистрибутив: {{ ansible_distribution }} {{ ansible_distribution_version }}
Время генерации: {{ ansible_date_time.iso8601 }}

{% for key in filtered_keys %}
Ключ: {{ key.name }}
URL: {{ key.url }}
Файл: {{ keys_destination }}/{{ key.dest | default(key.url | regex_replace('^.*/([^/]+)$', '\\1') }}
{% if key.checksum is defined %}Контрольная сумма: {{ key.checksum }}{% endif %}

{% endfor %}
{% if invalid_keys is defined and invalid_keys | length > 0 %}
ВНИМАНИЕ: Невалидные ключи:
{% for key in invalid_keys %}
- {{ key }}
{% endfor %}
{% endif %}
```

## Использование роли

1. В playbook можно переопределять переменные:

```yaml
- name: Установка ключей репозиториев
  hosts: all
  become: yes
  vars:
    repo_keys_dir_structure: "by_repo"
    repo_keys_to_download:
      - name: "NVIDIA"
        url: "https://developer.download.nvidia.com/compute/cuda/repos/{{ ansible_distribution | lower }}{{ ansible_distribution_major_version }}/x86_64/3bf863cc.pub"
        repos: ["nvidia"]
  roles:
    - repo_keys
```

2. Для конкретного дистрибутива можно добавить ключи через host_vars или group_vars.

## Преимущества новой версии:

1. **Гибкость**: Работает с любыми дистрибутивами Linux
2. **Настраиваемость**: Структура каталогов и источники ключей настраиваются через переменные
3. **Безопасность**: Проверка контрольных сумм и валидности ключей
4. **Масштабируемость**: Легко добавлять новые ключи без изменения роли
5. **Отказоустойчивость**: Повторные попытки загрузки и резервное копирование

Роль автоматически адаптируется под тип ОС (Debian/RedHat/другие) и создает соответствующую структуру каталогов.