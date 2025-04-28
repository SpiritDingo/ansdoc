# Роль Ansible для установки, настройки и управления Nginx

Эта роль Ansible позволяет автоматизировать установку, настройку и управление веб-сервером Nginx.

## Структура роли

Стандартная структура роли:

```
nginx-role/
├── defaults/
│   └── main.yml       # Умолчательные переменные
├── files/             # Статические файлы для копирования
├── handlers/
│   └── main.yml       # Обработчики (например, перезагрузка nginx)
├── meta/
│   └── main.yml       # Метаданные роли
├── tasks/
│   └── main.yml       # Основные задачи
├── templates/         # Шаблоны конфигурации (Jinja2)
└── vars/
    └── main.yml       # Переменные роли
```

## Основные файлы роли

### defaults/main.yml

```yaml
---
# Умолчательные переменные
nginx_packages:
  - nginx

nginx_user: www-data
nginx_worker_processes: "auto"
nginx_worker_connections: 1024
nginx_keepalive_timeout: 65
nginx_sendfile: "on"
nginx_tcp_nopush: "on"
nginx_tcp_nodelay: "on"
nginx_server_tokens: "off"
nginx_types_hash_max_size: 2048

nginx_conf_path: "/etc/nginx/nginx.conf"
nginx_sites_available: "/etc/nginx/sites-available"
nginx_sites_enabled: "/etc/nginx/sites-enabled"

nginx_remove_default_config: true
nginx_default_server: false

nginx_servers: []
# Пример структуры для nginx_servers:
# - name: example.com
#   listen: 80
#   server_name: example.com
#   root: /var/www/example.com
#   index: index.html index.htm
#   locations:
#     - path: /
#       try_files: "$uri $uri/ =404"
#   ssl:
#     enabled: false
#     cert: /path/to/cert
#     key: /path/to/key
#     listen: 443
```

### tasks/main.yml

```yaml
---
- name: Install Nginx
  apt:
    name: "{{ nginx_packages }}"
    state: present
    update_cache: yes
  when: ansible_os_family == 'Debian'
  
- name: Install Nginx (RedHat)
  yum:
    name: "{{ nginx_packages }}"
    state: present
  when: ansible_os_family == 'RedHat'

- name: Ensure Nginx is running and enabled
  service:
    name: nginx
    state: started
    enabled: yes

- name: Configure Nginx main configuration
  template:
    src: nginx.conf.j2
    dest: "{{ nginx_conf_path }}"
    owner: root
    group: root
    mode: 0644
  notify: restart nginx

- name: Remove default Nginx configuration if specified
  file:
    path: "{{ nginx_sites_enabled }}/default"
    state: absent
  when: nginx_remove_default_config
  notify: restart nginx

- name: Create Nginx server configurations
  template:
    src: server.conf.j2
    dest: "{{ nginx_sites_available }}/{{ item.name }}.conf"
    owner: root
    group: root
    mode: 0644
  with_items: "{{ nginx_servers }}"
  notify: restart nginx

- name: Enable Nginx server configurations
  file:
    src: "{{ nginx_sites_available }}/{{ item.name }}.conf"
    dest: "{{ nginx_sites_enabled }}/{{ item.name }}.conf"
    state: link
  with_items: "{{ nginx_servers }}"
  notify: restart nginx
```

### handlers/main.yml

```yaml
---
- name: restart nginx
  service:
    name: nginx
    state: restarted

- name: reload nginx
  service:
    name: nginx
    state: reloaded
```

### templates/nginx.conf.j2

```jinja2
user {{ nginx_user }};
worker_processes {{ nginx_worker_processes }};

error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections }};
}

http {
    sendfile {{ nginx_sendfile }};
    tcp_nopush {{ nginx_tcp_nopush }};
    tcp_nodelay {{ nginx_tcp_nodelay }};
    keepalive_timeout {{ nginx_keepalive_timeout }};
    types_hash_max_size {{ nginx_types_hash_max_size }};
    server_tokens {{ nginx_server_tokens }};

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    access_log /var/log/nginx/access.log;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### templates/server.conf.j2

```jinja2
server {
    listen {{ item.listen }};
    {% if item.server_name is defined %}
    server_name {{ item.server_name }};
    {% endif %}

    {% if item.ssl.enabled is defined and item.ssl.enabled %}
    listen {{ item.ssl.listen }} ssl;
    ssl_certificate {{ item.ssl.cert }};
    ssl_certificate_key {{ item.ssl.key }};
    {% endif %}

    root {{ item.root }};
    index {{ item.index | default('index.html index.htm') }};

    {% for location in item.locations %}
    location {{ location.path }} {
        {% if location.try_files is defined %}
        try_files {{ location.try_files }};
        {% endif %}
        {% if location.proxy_pass is defined %}
        proxy_pass {{ location.proxy_pass }};
        {% endif %}
    }
    {% endfor %}
}
```

## Использование роли

1. Добавьте роль в ваш проект Ansible (например, в `roles/nginx`)

2. Создайте playbook для использования роли:

```yaml
---
- hosts: webservers
  become: yes
  roles:
    - nginx
  vars:
    nginx_servers:
      - name: my_site
        listen: 80
        server_name: example.com
        root: /var/www/example.com
        index: index.html
        locations:
          - path: /
            try_files: "$uri $uri/ =404"
```

3. Для настройки SSL:

```yaml
nginx_servers:
  - name: my_ssl_site
    listen: 80
    server_name: example.com
    root: /var/www/example.com
    index: index.html
    ssl:
      enabled: true
      cert: /etc/ssl/certs/example.com.crt
      key: /etc/ssl/private/example.com.key
      listen: 443
    locations:
      - path: /
        try_files: "$uri $uri/ =404"
```

## Дополнительные возможности

1. **Управление виртуальными хостами**: Роль поддерживает создание и настройку множества виртуальных хостов.

2. **Поддержка SSL**: Можно легко настроить HTTPS для любого сайта.

3. **Гибкая настройка location**: Поддержка различных конфигураций для разных URL-путей.

4. **Кросс-платформенность**: Поддержка как Debian-based, так и RedHat-based систем.

5. **Безопасность**: По умолчанию отключены server tokens для повышения безопасности.

Эта роль предоставляет базовую функциональность для работы с Nginx, которую можно расширять в зависимости от конкретных потребностей.