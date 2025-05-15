# Роль Ansible для переноса настроек Nginx с старого сервера на новый

Вот полная роль Ansible для миграции конфигурации Nginx между серверами.

## Структура роли

```
roles/nginx_migration/
├── tasks/
│   ├── main.yml
│   ├── transfer_configs.yml
│   └── setup_nginx.yml
├── handlers/
│   └── main.yml
├── templates/
│   └── nginx.conf.j2
└── vars/
    └── main.yml
```

## Файлы роли

### roles/nginx_migration/tasks/main.yml

```yaml
---
- name: Include tasks for Nginx configuration transfer
  include_tasks: transfer_configs.yml

- name: Include tasks for Nginx setup
  include_tasks: setup_nginx.yml
```

### roles/nginx_migration/tasks/transfer_configs.yml

```yaml
---
- name: Ensure temporary directory exists
  ansible.builtin.file:
    path: /tmp/nginx_config_backup
    state: directory
    mode: '0755'

- name: Copy Nginx configuration from old server
  ansible.builtin.fetch:
    src: "{{ item }}"
    dest: /tmp/nginx_config_backup/
    flat: yes
  with_items:
    - /etc/nginx/nginx.conf
    - /etc/nginx/conf.d/
    - /etc/nginx/sites-available/
    - /etc/nginx/modules-available/

- name: Copy SSL certificates if exist
  ansible.builtin.fetch:
    src: /etc/nginx/ssl/
    dest: /tmp/nginx_config_backup/
    flat: yes
  ignore_errors: yes
  when: ssl_enabled | default(false)

- name: Copy custom error pages if exist
  ansible.builtin.fetch:
    src: /usr/share/nginx/html/errors/
    dest: /tmp/nginx_config_backup/
    flat: yes
  ignore_errors: yes
```

### roles/nginx_migration/tasks/setup_nginx.yml

```yaml
---
- name: Install Nginx
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: yes

- name: Ensure Nginx directories exist on new server
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  with_items:
    - /etc/nginx/sites-available
    - /etc/nginx/sites-enabled
    - /etc/nginx/conf.d
    - /etc/nginx/modules-available
    - /etc/nginx/ssl
    - /usr/share/nginx/html/errors

- name: Deploy main Nginx configuration
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    mode: '0644'
  notify: reload nginx

- name: Copy configuration files
  ansible.builtin.copy:
    src: "/tmp/nginx_config_backup/{{ item }}"
    dest: "/etc/nginx/{{ item }}"
    remote_src: yes
    mode: '0644'
  with_items:
    - conf.d/
    - sites-available/
    - modules-available/
  notify: reload nginx

- name: Enable sites
  ansible.builtin.file:
    src: "/etc/nginx/sites-available/{{ item }}"
    dest: "/etc/nginx/sites-enabled/{{ item }}"
    state: link
  with_items: "{{ nginx_sites }}"
  notify: reload nginx

- name: Copy SSL certificates
  ansible.builtin.copy:
    src: "/tmp/nginx_config_backup/ssl/"
    dest: "/etc/nginx/ssl/"
    remote_src: yes
    mode: '0600'
  when: ssl_enabled | default(false)
  notify: reload nginx

- name: Copy custom error pages
  ansible.builtin.copy:
    src: "/tmp/nginx_config_backup/errors/"
    dest: "/usr/share/nginx/html/errors/"
    remote_src: yes
  notify: reload nginx
```

### roles/nginx_migration/handlers/main.yml

```yaml
---
- name: reload nginx
  ansible.builtin.service:
    name: nginx
    state: reloaded

- name: restart nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```

### roles/nginx_migration/vars/main.yml

```yaml
---
nginx_sites:
  - default
  # Добавьте другие конфиги сайтов, которые нужно включить

ssl_enabled: false
# Установите true, если используется SSL

nginx_worker_processes: "auto"
nginx_worker_connections: 1024
```

### templates/nginx.conf.j2

```nginx
user www-data;
worker_processes {{ nginx_worker_processes }};
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections {{ nginx_worker_connections }};
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    gzip on;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

## Пример playbook для использования роли

```yaml
---
- name: Migrate Nginx configuration
  hosts: new_nginx_servers
  become: yes
  vars:
    source_server: old_nginx_server  # Хост из inventory старого сервера
  pre_tasks:
    - name: Gather Nginx facts from old server
      delegate_to: "{{ source_server }}"
      ansible.builtin.shell: |
        echo "nginx_sites:"
        ls /etc/nginx/sites-available | awk '{print "  - " $1}'
        echo "ssl_enabled: $(test -d /etc/nginx/ssl && echo true || echo false)"
      register: nginx_facts

    - name: Set Nginx variables from old server
      ansible.builtin.set_fact:
        nginx_sites: "{{ (nginx_facts.stdout | from_yaml).nginx_sites }}"
        ssl_enabled: "{{ (nginx_facts.stdout | from_yaml).ssl_enabled }}"

  roles:
    - nginx_migration

  post_tasks:
    - name: Test Nginx configuration
      ansible.builtin.command: nginx -t
      register: nginx_test
      changed_when: false

    - name: Show Nginx test result
      ansible.builtin.debug:
        var: nginx_test.stdout_lines
```

## Дополнительные рекомендации

1. **Проверка зависимостей**:
   ```yaml
   - name: Check for required modules
     ansible.builtin.stat:
       path: "/etc/nginx/modules-available/{{ item }}.conf"
     with_items: "{{ nginx_modules }}"
     register: modules_check
     delegate_to: "{{ source_server }}"
   ```

2. **Перенос специфичных настроек**:
   ```yaml
   - name: Transfer custom configurations
     ansible.builtin.fetch:
       src: "{{ item }}"
       dest: /tmp/nginx_config_backup/
       flat: yes
     with_items:
       - /etc/nginx/fastcgi_params
       - /etc/nginx/uwsgi_params
       - /etc/nginx/scgi_params
     ignore_errors: yes
   ```

3. **Настройка прав**:
   ```yaml
   - name: Set proper permissions
     ansible.builtin.file:
       path: "{{ item.path }}"
       owner: "{{ item.owner }}"
       group: "{{ item.group }}"
       mode: "{{ item.mode }}"
     with_items:
       - { path: "/etc/nginx/ssl", owner: "root", group: "root", mode: "0700" }
       - { path: "/etc/nginx/ssl/*.key", owner: "root", group: "root", mode: "0600" }
   ```

4. **Логирование**:
   ```yaml
   - name: Configure log rotation
     ansible.builtin.copy:
       src: /tmp/nginx_config_backup/logrotate.d/nginx
       dest: /etc/logrotate.d/nginx
       remote_src: yes
   ```

Для сложных конфигураций рекомендуется предварительно проанализировать старую конфигурацию и адаптировать переменные в vars/main.yml под ваши нужды.