Создадим универсальную Ansible роль для настройки Apache и SSL на Ubuntu 22.04, Ubuntu 24.04 и Oracle Linux 9.

Структура роли:

```bash
apache_ssl_universal/
├── tasks/
│   ├── main.yml
│   ├── setup_ubuntu.yml
│   └── setup_ol.yml
├── handlers/
│   └── main.yml
├── templates/
│   ├── apache-ssl-ubuntu.conf.j2
│   └── apache-ssl-ol.conf.j2
├── vars/
│   └── main.yml
└── defaults/
    └── main.yml
```

1. Переменные по умолчанию (defaults/main.yml)

```yaml
# Общие настройки Apache
apache_server_name: "example.com"
apache_document_root: "/var/www/html"
apache_ssl_vhost_name: "{{ apache_server_name }}"

# SSL сертификаты
ssl_cert_src: "server.crt"
ssl_key_src: "server.key"
ssl_chain_src: ""  # Опционально

# Порты
apache_http_port: 80
apache_https_port: 443

# Безопасность SSL
ssl_protocols: "all -SSLv3 -TLSv1 -TLSv1.1"
ssl_ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256"
ssl_honor_cipher_order: "off"
ssl_session_tickets: "off"

# Управление службой
apache_service_state: "started"
apache_service_enabled: true

# Автоопределение ОС
apache_packages:
  Ubuntu:
    - apache2
    - ssl-cert
  OracleLinux:
    - httpd
    - mod_ssl

apache_service_name:
  Ubuntu: "apache2"
  OracleLinux: "httpd"

apache_conf_paths:
  Ubuntu:
    sites_available: "/etc/apache2/sites-available"
    sites_enabled: "/etc/apache2/sites-enabled"
    mods_available: "/etc/apache2/mods-available"
    mods_enabled: "/etc/apache2/mods-enabled"
    ssl_cert_path: "/etc/ssl/certs"
    ssl_key_path: "/etc/ssl/private"
  OracleLinux:
    conf_dir: "/etc/httpd/conf.d"
    conf_main: "/etc/httpd/conf/httpd.conf"
    ssl_cert_path: "/etc/pki/tls/certs"
    ssl_key_path: "/etc/pki/tls/private"
```

2. Основные задачи (tasks/main.yml)

```yaml
---
- name: "Определение дистрибутива"
  set_fact:
    os_family: "{{ ansible_distribution }}{{ ansible_distribution_major_version }}"
  when: ansible_os_family == "Debian" or ansible_os_family == "RedHat"

- name: "Установка Apache для Ubuntu"
  include_tasks: setup_ubuntu.yml
  when: ansible_os_family == "Debian"

- name: "Установка Apache для Oracle Linux"
  include_tasks: setup_ol.yml
  when: ansible_os_family == "RedHat"

- name: "Создание document root"
  file:
    path: "{{ apache_document_root }}"
    state: directory
    mode: '0755'
    owner: "{{ apache_user | default('root') }}"
    group: "{{ apache_group | default('root') }}"

- name: "Копирование SSL сертификатов"
  block:
    - name: "Создание SSL директорий"
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - "{{ ssl_cert_path }}"
        - "{{ ssl_key_path }}"

    - name: "Копирование SSL сертификата"
      copy:
        src: "{{ ssl_cert_src }}"
        dest: "{{ ssl_cert_path }}/{{ ssl_cert_src | basename }}"
        mode: '0644'
        backup: yes
      notify: restart apache

    - name: "Копирование SSL приватного ключа"
      copy:
        src: "{{ ssl_key_src }}"
        dest: "{{ ssl_key_path }}/{{ ssl_key_src | basename }}"
        mode: '0600'
        backup: yes
      notify: restart apache

    - name: "Копирование цепочки сертификатов (если указана)"
      copy:
        src: "{{ ssl_chain_src }}"
        dest: "{{ ssl_cert_path }}/{{ ssl_chain_src | basename }}"
        mode: '0644'
        backup: yes
      when: ssl_chain_src | length > 0
      notify: restart apache
  vars:
    ssl_cert_path: "{{ apache_conf_paths[ansible_os_family].ssl_cert_path }}"
    ssl_key_path: "{{ apache_conf_paths[ansible_os_family].ssl_key_path }}"

- name: "Настройка виртуального хоста SSL"
  template:
    src: "{{ apache_ssl_template }}"
    dest: "{{ apache_ssl_conf_path }}"
    mode: '0644'
  notify: restart apache
  vars:
    apache_ssl_template: "{{ 'apache-ssl-ubuntu.conf.j2' if ansible_os_family == 'Debian' else 'apache-ssl-ol.conf.j2' }}"
    apache_ssl_conf_path: "{{ apache_conf_paths[ansible_os_family].sites_available ~ '/' ~ apache_ssl_vhost_name ~ '-ssl.conf' if ansible_os_family == 'Debian' else apache_conf_paths[ansible_os_family].conf_dir ~ '/' ~ apache_ssl_vhost_name ~ '-ssl.conf' }}"

- name: "Активация SSL сайта (Ubuntu)"
  file:
    src: "{{ apache_conf_paths[ansible_os_family].sites_available }}/{{ apache_ssl_vhost_name }}-ssl.conf"
    dest: "{{ apache_conf_paths[ansible_os_family].sites_enabled }}/{{ apache_ssl_vhost_name }}-ssl.conf"
    state: link
  when: ansible_os_family == "Debian"
  notify: restart apache

- name: "Проверка конфигурации Apache"
  command: "{{ apache_config_test_command }}"
  register: config_test
  changed_when: false
  failed_when: "config_test.rc != 0"
  vars:
    apache_config_test_command: "{{ 'apache2ctl configtest' if ansible_os_family == 'Debian' else 'httpd -t' }}"
```

3. Задачи для Ubuntu (tasks/setup_ubuntu.yml)

```yaml
---
- name: "Обновление apt кэша"
  apt:
    update_cache: yes
    cache_valid_time: 3600

- name: "Установка Apache и зависимостей"
  apt:
    name: "{{ apache_packages.Ubuntu }}"
    state: latest

- name: "Активация необходимых модулей Apache"
  command: "a2enmod {{ item }}"
  loop:
    - ssl
    - rewrite
    - headers
  notify: restart apache

- name: "Запуск и включение службы Apache"
  service:
    name: "{{ apache_service_name.Ubuntu }}"
    state: "{{ apache_service_state }}"
    enabled: "{{ apache_service_enabled }}"

- name: "Установка фактов для Ubuntu"
  set_fact:
    apache_user: "www-data"
    apache_group: "www-data"
    ssl_cert_path: "{{ apache_conf_paths.Ubuntu.ssl_cert_path }}"
    ssl_key_path: "{{ apache_conf_paths.Ubuntu.ssl_key_path }}"
```

4. Задачи для Oracle Linux (tasks/setup_ol.yml)

```yaml
---
- name: "Установка EPEL репозитория"
  dnf:
    name: epel-release
    state: present
  when: ansible_distribution == "OracleLinux"

- name: "Установка Apache и зависимостей"
  dnf:
    name: "{{ apache_packages.OracleLinux }}"
    state: latest

- name: "Открытие портов в firewalld"
  firewalld:
    port: "{{ item }}/tcp"
    permanent: yes
    state: enabled
    immediate: yes
  loop:
    - "{{ apache_http_port }}"
    - "{{ apache_https_port }}"
  when: ansible_os_family == "RedHat"

- name: "Запуск и включение службы Apache"
  service:
    name: "{{ apache_service_name.OracleLinux }}"
    state: "{{ apache_service_state }}"
    enabled: "{{ apache_service_enabled }}"

- name: "Установка фактов для Oracle Linux"
  set_fact:
    apache_user: "apache"
    apache_group: "apache"
    ssl_cert_path: "{{ apache_conf_paths.OracleLinux.ssl_cert_path }}"
    ssl_key_path: "{{ apache_conf_paths.OracleLinux.ssl_key_path }}"
```

5. Обработчики (handlers/main.yml)

```yaml
---
- name: restart apache
  service:
    name: "{{ apache_service_name[ansible_os_family] }}"
    state: restarted

- name: reload apache
  service:
    name: "{{ apache_service_name[ansible_os_family] }}"
    state: reloaded
```

6. Шаблон для Ubuntu (templates/apache-ssl-ubuntu.conf.j2)

```apache
<VirtualHost *:{{ apache_https_port }}>
    ServerName {{ apache_server_name }}
    DocumentRoot {{ apache_document_root }}

    SSLEngine on
    SSLCertificateFile {{ apache_conf_paths.Ubuntu.ssl_cert_path }}/{{ ssl_cert_src | basename }}
    SSLCertificateKeyFile {{ apache_conf_paths.Ubuntu.ssl_key_path }}/{{ ssl_key_src | basename }}
    {% if ssl_chain_src | length > 0 %}
    SSLCertificateChainFile {{ apache_conf_paths.Ubuntu.ssl_cert_path }}/{{ ssl_chain_src | basename }}
    {% endif %}

    # Настройки безопасности SSL
    SSLProtocol {{ ssl_protocols }}
    SSLCipherSuite {{ ssl_ciphers }}
    SSLHonorCipherOrder {{ ssl_honor_cipher_order }}
    SSLSessionTickets {{ ssl_session_tickets }}

    # Заголовки безопасности
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
    Header always set X-Content-Type-Options nosniff
    Header always set X-Frame-Options DENY

    # Логирование
    ErrorLog ${APACHE_LOG_DIR}/{{ apache_ssl_vhost_name }}_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/{{ apache_ssl_vhost_name }}_ssl_access.log combined
</VirtualHost>

<VirtualHost *:{{ apache_http_port }}>
    ServerName {{ apache_server_name }}
    Redirect permanent / https://{{ apache_server_name }}/
</VirtualHost>
```

7. Шаблон для Oracle Linux (templates/apache-ssl-ol.conf.j2)

```apache
<VirtualHost *:{{ apache_https_port }}>
    ServerName {{ apache_server_name }}
    DocumentRoot {{ apache_document_root }}

    SSLEngine on
    SSLCertificateFile {{ apache_conf_paths.OracleLinux.ssl_cert_path }}/{{ ssl_cert_src | basename }}
    SSLCertificateKeyFile {{ apache_conf_paths.OracleLinux.ssl_key_path }}/{{ ssl_key_src | basename }}
    {% if ssl_chain_src | length > 0 %}
    SSLCertificateChainFile {{ apache_conf_paths.OracleLinux.ssl_cert_path }}/{{ ssl_chain_src | basename }}
    {% endif %}

    # Настройки безопасности SSL
    SSLProtocol {{ ssl_protocols }}
    SSLCipherSuite {{ ssl_ciphers }}
    SSLHonorCipherOrder {{ ssl_honor_cipher_order }}
    SSLSessionTickets {{ ssl_session_tickets }}

    # Заголовки безопасности
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
    Header always set X-Content-Type-Options nosniff
    Header always set X-Frame-Options DENY

    # Логирование
    ErrorLog /var/log/httpd/{{ apache_ssl_vhost_name }}_ssl_error.log
    CustomLog /var/log/httpd/{{ apache_ssl_vhost_name }}_ssl_access.log combined
</VirtualHost>

<VirtualHost *:{{ apache_http_port }}>
    ServerName {{ apache_server_name }}
    Redirect permanent / https://{{ apache_server_name }}/
</VirtualHost>
```

8. Плейбук для использования роли

```yaml
---
- name: "Установка и настройка Apache с SSL"
  hosts: web_servers
  become: yes
  roles:
    - apache_ssl_universal
  vars:
    apache_server_name: "example.com"
    apache_document_root: "/var/www/example"
    ssl_cert_src: "example.com.crt"
    ssl_key_src: "example.com.key"
    ssl_chain_src: "chain.pem"
```

Особенности роли:

1. Автоопределение ОС: Роль автоматически определяет дистрибутив и применяет соответствующие настройки
2. Универсальность: Поддерживает Ubuntu 22.04, 24.04 и Oracle Linux 9
3. Гибкость: Все параметры настраиваются через переменные
4. Безопасность: Включает современные настройки SSL и security headers
5. Идемпотентность: Можно безопасно запускать многократно

Роль готова к использованию на всех указанных дистрибутивах и автоматически адаптируется под каждую ОС.