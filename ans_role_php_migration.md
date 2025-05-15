# Роль Ansible для переноса настроек PHP с старого сервера на новый

Вот полная роль Ansible для миграции конфигурации PHP между серверами.

## Структура роли

```
roles/php_migration/
├── tasks/
│   ├── main.yml
│   ├── transfer_configs.yml
│   ├── setup_php.yml
│   └── validate_php.yml
├── handlers/
│   └── main.yml
├── templates/
│   ├── php.ini.j2
│   └── www.conf.j2
└── vars/
    └── main.yml
```

## Файлы роли

### roles/php_migration/tasks/main.yml

```yaml
---
- name: Gather PHP facts from old server
  delegate_to: "{{ source_server }}"
  ansible.builtin.shell: |
    php -v | head -n 1
    php -i | grep "Loaded Configuration File"
    php -m
  register: php_facts_old
  changed_when: false

- name: Set PHP version fact
  ansible.builtin.set_fact:
    php_version: "{{ php_facts_old.stdout_lines[0] | regex_search('PHP\\s([0-9]+\\.[0-9]+)', '\\1') }}"

- name: Include transfer tasks
  include_tasks: transfer_configs.yml

- name: Include setup tasks
  include_tasks: setup_php.yml

- name: Include validation tasks
  include_tasks: validate_php.yml
```

### roles/php_migration/tasks/transfer_configs.yml

```yaml
---
- name: Create temporary directory
  ansible.builtin.file:
    path: /tmp/php_config_backup
    state: directory
    mode: '0755'

- name: Copy PHP configuration files
  ansible.builtin.fetch:
    src: "{{ item }}"
    dest: /tmp/php_config_backup/
    flat: yes
  with_items:
    - /etc/php/{{ php_version }}/cli/php.ini
    - /etc/php/{{ php_version }}/fpm/php.ini
    - /etc/php/{{ php_version }}/fpm/pool.d/www.conf
    - /etc/php/{{ php_version }}/mods-available/

- name: Copy custom PHP modules
  ansible.builtin.fetch:
    src: /usr/lib/php/{{ php_version }}/
    dest: /tmp/php_config_backup/modules/
    flat: yes
  ignore_errors: yes
```

### roles/php_migration/tasks/setup_php.yml

```yaml
---
- name: Install PHP
  ansible.builtin.apt:
    name:
      - php{{ php_version }}
      - php{{ php_version }}-fpm
      - php{{ php_version }}-cli
      - php{{ php_version }}-common
    state: present
    update_cache: yes

- name: Install additional PHP modules
  ansible.builtin.apt:
    name: "php{{ php_version }}-{{ item }}"
    state: present
  with_items: "{{ php_modules }}"

- name: Deploy main PHP configuration
  ansible.builtin.template:
    src: php.ini.j2
    dest: /etc/php/{{ php_version }}/fpm/php.ini
    mode: '0644'
  notify: restart php

- name: Deploy FPM pool configuration
  ansible.builtin.template:
    src: www.conf.j2
    dest: /etc/php/{{ php_version }}/fpm/pool.d/www.conf
    mode: '0644'
  notify: restart php

- name: Copy PHP modules configuration
  ansible.builtin.copy:
    src: "/tmp/php_config_backup/mods-available/"
    dest: "/etc/php/{{ php_version }}/mods-available/"
    remote_src: yes
    mode: '0644'
  notify: restart php

- name: Enable PHP modules
  ansible.builtin.file:
    src: "/etc/php/{{ php_version }}/mods-available/{{ item }}.ini"
    dest: "/etc/php/{{ php_version }}/fpm/conf.d/20-{{ item }}.ini"
    state: link
  with_items: "{{ php_modules }}"
  notify: restart php

- name: Copy custom PHP modules
  ansible.builtin.copy:
    src: "/tmp/php_config_backup/modules/"
    dest: "/usr/lib/php/{{ php_version }}/"
    remote_src: yes
  when: custom_php_modules | default(false)
  notify: restart php
```

### roles/php_migration/tasks/validate_php.yml

```yaml
---
- name: Validate PHP-FPM configuration
  ansible.builtin.command: php-fpm{{ php_version }} -t
  register: php_validation
  changed_when: false
  ignore_errors: yes

- name: Show validation results
  ansible.builtin.debug:
    var: php_validation.stdout_lines

- name: Test PHP configuration
  ansible.builtin.command: php -i
  register: php_test
  changed_when: false

- name: Show loaded PHP modules
  ansible.builtin.debug:
    msg: "PHP modules loaded: {{ php_test.stdout | regex_findall('(?<=^\\[)\\w+(?=\\])') }}"
```

### roles/php_migration/handlers/main.yml

```yaml
---
- name: restart php
  ansible.builtin.service:
    name: "php{{ php_version }}-fpm"
    state: restarted
```

### roles/php_migration/vars/main.yml

```yaml
---
php_modules:
  - mysql
  - xml
  - mbstring
  - curl
  - gd
  - zip
  # Добавьте другие модули, которые используются

php_ini_settings:
  memory_limit: "256M"
  upload_max_filesize: "64M"
  post_max_size: "128M"
  max_execution_time: "300"
  date.timezone: "UTC"
  # Добавьте другие настройки php.ini

fpm_settings:
  pm: "dynamic"
  pm.max_children: "50"
  pm.start_servers: "5"
  pm.min_spare_servers: "5"
  pm.max_spare_servers: "10"
  # Добавьте другие настройки PHP-FPM
```

### templates/php.ini.j2

```ini
; Ansible managed - DO NOT EDIT MANUALLY

[PHP]
engine = On
short_open_tag = Off
precision = 14
output_buffering = 4096
zlib.output_compression = Off
implicit_flush = Off
unserialize_callback_func =
serialize_precision = -1

{% for key, value in php_ini_settings.items() %}
{{ key }} = {{ value }}
{% endfor %}

; Other configurations from original php.ini
...
```

### templates/www.conf.j2

```ini
; Ansible managed - DO NOT EDIT MANUALLY

[www]
user = www-data
group = www-data
listen = /run/php/php{{ php_version }}-fpm.sock
listen.owner = www-data
listen.group = www-data
listen.mode = 0660

{% for key, value in fpm_settings.items() %}
{{ key }} = {{ value }}
{% endfor %}

; Other pool configurations
...
```

## Пример playbook для использования роли

```yaml
---
- name: Migrate PHP configuration
  hosts: new_php_servers
  become: yes
  vars:
    source_server: old_php_server  # Хост из inventory старого сервера
    
  pre_tasks:
    - name: Detect PHP modules on old server
      delegate_to: "{{ source_server }}"
      ansible.builtin.shell: |
        php -m | grep -v '^\\[' | grep -v '^Core$'
      register: old_php_modules
      changed_when: false

    - name: Set PHP modules fact
      ansible.builtin.set_fact:
        php_modules: "{{ old_php_modules.stdout_lines }}"

  roles:
    - php_migration

  post_tasks:
    - name: Verify PHP version
      ansible.builtin.command: php -v
      register: php_version_check
      changed_when: false

    - name: Display PHP version
      ansible.builtin.debug:
        msg: "PHP version installed: {{ php_version_check.stdout_lines[0] }}"
```

## Дополнительные рекомендации

1. **Перенос специфичных настроек**:
   ```yaml
   - name: Transfer custom ini files
     ansible.builtin.fetch:
       src: /etc/php/{{ php_version }}/fpm/conf.d/*.ini
       dest: /tmp/php_config_backup/custom-ini/
       flat: no
     ignore_errors: yes
   ```

2. **Настройка прав**:
   ```yaml
   - name: Set proper permissions for PHP sessions
     ansible.builtin.file:
       path: /var/lib/php/sessions
       state: directory
       owner: www-data
       group: www-data
       mode: '1733'
   ```

3. **Оптимизация PHP-FPM**:
   ```yaml
   - name: Tune PHP-FPM settings based on available memory
     ansible.builtin.set_fact:
       fpm_settings:
         pm.max_children: "{{ (ansible_memtotal_mb * 0.8 / 64) | int }}"
         pm.start_servers: "{{ (ansible_memtotal_mb * 0.1 / 64) | int }}"
         pm.min_spare_servers: "{{ (ansible_memtotal_mb * 0.1 / 64) | int }}"
         pm.max_spare_servers: "{{ (ansible_memtotal_mb * 0.3 / 64) | int }}"
     when: auto_tune_fpm | default(true)
   ```

4. **Перенос custom.ini файлов**:
   ```yaml
   - name: Copy custom PHP ini files
     ansible.builtin.copy:
       src: "/tmp/php_config_backup/custom-ini/"
       dest: "/etc/php/{{ php_version }}/fpm/conf.d/"
       remote_src: yes
     notify: restart php
   ```

5. **Проверка совместимости**:
   ```yaml
   - name: Check for deprecated functions
     ansible.builtin.command: php -l /path/to/your/php/files
     register: php_lint
     changed_when: false
     ignore_errors: yes
   ```

Для сложных конфигураций рекомендуется:
1. Сначала провести аудит старой конфигурации PHP
2. Сравнить версии PHP на старом и новом серверах
3. Проверить совместимость всех модулей
4. Протестировать миграцию на staging-окружении перед применением на production