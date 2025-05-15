# Роль Ansible для переноса настроек Apache2 с старого сервера на новый

Вот пример роли Ansible для переноса конфигурации Apache2 с одного сервера на другой.

## Структура роли

```
roles/apache_migration/
├── tasks/
│   ├── main.yml
│   ├── transfer_configs.yml
│   └── setup_apache.yml
├── handlers/
│   └── main.yml
├── templates/
│   └── apache2.conf.j2
└── vars/
    └── main.yml
```

## Файлы роли

### roles/apache_migration/tasks/main.yml

```yaml
---
- name: Include tasks for Apache configuration transfer
  include_tasks: transfer_configs.yml

- name: Include tasks for Apache setup
  include_tasks: setup_apache.yml
```

### roles/apache_migration/tasks/transfer_configs.yml

```yaml
---
- name: Ensure temporary directory exists
  ansible.builtin.file:
    path: /tmp/apache_config_backup
    state: directory
    mode: '0755'

- name: Copy Apache configuration from old server
  ansible.builtin.fetch:
    src: "{{ item }}"
    dest: /tmp/apache_config_backup/
    flat: yes
  with_items:
    - /etc/apache2/apache2.conf
    - /etc/apache2/ports.conf
    - /etc/apache2/sites-available/
    - /etc/apache2/conf-available/
    - /etc/apache2/mods-available/

- name: Copy SSL certificates if exist
  ansible.builtin.fetch:
    src: /etc/apache2/ssl/
    dest: /tmp/apache_config_backup/
    flat: yes
  ignore_errors: yes
  when: "'ssl' in apache_modules"
```

### roles/apache_migration/tasks/setup_apache.yml

```yaml
---
- name: Install Apache2
  ansible.builtin.apt:
    name: apache2
    state: present
    update_cache: yes

- name: Ensure Apache directories exist on new server
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  with_items:
    - /etc/apache2/sites-available
    - /etc/apache2/conf-available
    - /etc/apache2/mods-available
    - /etc/apache2/ssl

- name: Deploy main Apache configuration
  ansible.builtin.template:
    src: apache2.conf.j2
    dest: /etc/apache2/apache2.conf
    mode: '0644'
  notify: restart apache

- name: Copy configuration files
  ansible.builtin.copy:
    src: "/tmp/apache_config_backup/{{ item }}"
    dest: "/etc/apache2/{{ item }}"
    remote_src: yes
    mode: '0644'
  with_items:
    - ports.conf
    - sites-available/
    - conf-available/
    - mods-available/
  notify: restart apache

- name: Copy SSL certificates
  ansible.builtin.copy:
    src: "/tmp/apache_config_backup/ssl/"
    dest: "/etc/apache2/ssl/"
    remote_src: yes
    mode: '0600'
  when: "'ssl' in apache_modules"
  notify: restart apache

- name: Enable sites
  ansible.builtin.command: a2ensite {{ item }}
  with_items: "{{ apache_sites }}"
  notify: restart apache

- name: Enable modules
  ansible.builtin.command: a2enmod {{ item }}
  with_items: "{{ apache_modules }}"
  notify: restart apache

- name: Enable configurations
  ansible.builtin.command: a2enconf {{ item }}
  with_items: "{{ apache_confs }}"
  notify: restart apache
```

### roles/apache_migration/handlers/main.yml

```yaml
---
- name: restart apache
  ansible.builtin.service:
    name: apache2
    state: restarted
    enabled: yes
```

### roles/apache_migration/vars/main.yml

```yaml
---
apache_modules:
  - rewrite
  - headers
  # Добавьте другие модули, которые используются на старом сервере

apache_sites:
  - 000-default
  # Добавьте другие сайты, которые нужно включить

apache_confs:
  # Добавьте конфигурации, которые нужно включить
```

## Пример playbook для использования роли

```yaml
---
- name: Migrate Apache configuration
  hosts: new_apache_servers
  become: yes
  vars:
    source_server: old_apache_server  # Хост из inventory старого сервера
  pre_tasks:
    - name: Gather Apache facts from old server
      delegate_to: "{{ source_server }}"
      ansible.builtin.setup:
        filter: ansible_local
      register: old_server_facts

    - name: Set Apache modules from old server
      ansible.builtin.set_fact:
        apache_modules: "{{ old_server_facts.ansible_facts.ansible_local.apache2.modules_enabled }}"
      when: "'apache2' in old_server_facts.ansible_facts.ansible_local"

  roles:
    - apache_migration
```

## Примечания

1. Перед выполнением убедитесь, что:
   - У вас есть SSH доступ к обоим серверам
   - На старом сервере установлен Apache2
   - На новом сервере достаточно места и совместимая версия ОС

2. Для более сложных конфигураций может потребоваться:
   - Перенос пользовательских скриптов
   - Перенос данных веб-сайтов
   - Настройка прав доступа
   - Адаптация конфигурации под новое окружение

3. Рекомендуется сначала протестировать миграцию на тестовом сервере.

4. Для работы с SSL сертификатами может потребоваться дополнительная настройка.