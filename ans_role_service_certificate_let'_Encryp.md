# Ansible роль для замены сертификатов на серверах

Роль обеспечивает централизованное управление SSL/TLS сертификатами на множестве серверов с поддержкой различных сервисов (Nginx, Apache, Postfix, Dovecot и др.).

## Структура роли

```
roles/certificate_replacement/
├── defaults/
│   └── main.yml          # Параметры по умолчанию
├── files/
│   └── example.crt       # Пример сертификата (опционально)
├── tasks/
│   ├── main.yml          # Основной плейбук
│   ├── nginx.yml         # Замена для Nginx
│   ├── apache.yml        # Замена для Apache
│   ├── postfix.yml       # Замена для Postfix
│   ├── dovecot.yml      # Замена для Dovecot
│   └── validate.yml     # Валидация сертификатов
├── templates/
│   ├── nginx.conf.j2    # Шаблон конфига Nginx
│   └── apache.conf.j2   # Шаблон конфига Apache
└── handlers/
    └── main.yml         # Обработчики перезагрузки сервисов
```

## Основные файлы роли

### defaults/main.yml

```yaml
---
# Основные параметры сертификатов
certificates:
  base_dir: "/etc/ssl/certs"          # Базовая директория для сертификатов
  private_key_dir: "/etc/ssl/private" # Директория для приватных ключей
  owner: "root"                       # Владелец файлов
  group: "root"                       # Группа файлов
  cert_mode: "0644"                   # Права на сертификаты
  key_mode: "0600"                    # Права на приватные ключи

# Сервисы, использующие сертификаты
services:
  nginx:
    enabled: false
    config_dir: "/etc/nginx/conf.d"
    cert_path: "{{ certificates.base_dir }}/nginx.crt"
    key_path: "{{ certificates.private_key_dir }}/nginx.key"
  
  apache:
    enabled: false
    config_dir: "/etc/httpd/conf.d"
    cert_path: "{{ certificates.base_dir }}/apache.crt"
    key_path: "{{ certificates.private_key_dir }}/apache.key"

  postfix:
    enabled: false
    cert_path: "{{ certificates.base_dir }}/postfix.crt"
    key_path: "{{ certificates.private_key_dir }}/postfix.key"

  dovecot:
    enabled: false
    cert_path: "{{ certificates.base_dir }}/dovecot.crt"
    key_path: "{{ certificates.private_key_dir }}/dovecot.key"

# Валидация сертификатов
validation:
  check_expiration: true
  days_before_expire: 30
  verify_chain: true
```

### tasks/main.yml

```yaml
---
- name: Ensure SSL directories exist
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    owner: "{{ certificates.owner }}"
    group: "{{ certificates.group }}"
    mode: "0755"
  loop:
    - "{{ certificates.base_dir }}"
    - "{{ certificates.private_key_dir }}"

- name: Deploy new certificates
  ansible.builtin.copy:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    owner: "{{ certificates.owner }}"
    group: "{{ certificates.group }}"
    mode: "{{ item.mode }}"
    remote_src: false
  loop: "{{ certificate_files }}"
  when: certificate_files is defined

- name: Include Nginx tasks if enabled
  ansible.builtin.include_tasks: nginx.yml
  when: services.nginx.enabled

- name: Include Apache tasks if enabled
  ansible.builtin.include_tasks: apache.yml
  when: services.apache.enabled

- name: Include Postfix tasks if enabled
  ansible.builtin.include_tasks: postfix.yml
  when: services.postfix.enabled

- name: Include Dovecot tasks if enabled
  ansible.builtin.include_tasks: dovecot.yml
  when: services.dovecot.enabled

- name: Validate certificates
  ansible.builtin.include_tasks: validate.yml
```

### tasks/nginx.yml

```yaml
---
- name: Configure Nginx SSL
  ansible.builtin.template:
    src: "nginx.conf.j2"
    dest: "{{ services.nginx.config_dir }}/ssl.conf"
    owner: "{{ certificates.owner }}"
    group: "{{ certificates.group }}"
    mode: "0644"
  notify: Reload Nginx

- name: Verify Nginx configuration
  ansible.builtin.command: nginx -t
  changed_when: false
  register: nginx_test
  failed_when: nginx_test.rc != 0
```

### tasks/apache.yml

```yaml
---
- name: Configure Apache SSL
  ansible.builtin.template:
    src: "apache.conf.j2"
    dest: "{{ services.apache.config_dir }}/ssl.conf"
    owner: "{{ certificates.owner }}"
    group: "{{ certificates.group }}"
    mode: "0644"
  notify: Restart Apache

- name: Verify Apache configuration
  ansible.builtin.command: apachectl configtest
  changed_when: false
  register: apache_test
  failed_when: apache_test.rc != 0
```

### tasks/validate.yml

```yaml
---
- name: Check certificate expiration
  ansible.builtin.command: >
    openssl x509 -enddate -noout -in {{ item.cert_path }}
  register: cert_check
  changed_when: false
  loop: "{{ certs_to_validate }}"
  when: validation.check_expiration

- name: Verify certificate chain
  ansible.builtin.command: >
    openssl verify -CAfile {{ ca_bundle_path }} {{ item.cert_path }}
  register: chain_verify
  changed_when: false
  loop: "{{ certs_to_validate }}"
  when: validation.verify_chain and ca_bundle_path is defined
```

### handlers/main.yml

```yaml
---
- name: Reload Nginx
  ansible.builtin.service:
    name: nginx
    state: reloaded

- name: Restart Apache
  ansible.builtin.service:
    name: apache2
    state: restarted

- name: Restart Postfix
  ansible.builtin.service:
    name: postfix
    state: restarted

- name: Restart Dovecot
  ansible.builtin.service:
    name: dovecot
    state: restarted
```

## Пример использования роли

1. Создайте playbook `replace_certs.yml`:

```yaml
---
- hosts: web_servers
  become: yes
  vars:
    certificate_files:
      - src: "files/new_cert.crt"
        dest: "/etc/ssl/certs/server.crt"
        mode: "0644"
      - src: "files/new_key.key"
        dest: "/etc/ssl/private/server.key"
        mode: "0600"
    
    services:
      nginx:
        enabled: true
      apache:
        enabled: false
    
    ca_bundle_path: "/etc/ssl/certs/ca-bundle.crt"

  roles:
    - certificate_replacement
```

2. Подготовьте файлы сертификатов в директории `files/` вашего playbook.

3. Запустите playbook:

```bash
ansible-playbook -i inventory.ini replace_certs.yml
```

## Дополнительные возможности

1. **Автоматическое обновление сертификатов**:
   - Интеграция с Let's Encrypt для автоматического получения сертификатов
   - Поддержка certbot для автоматического обновления

2. **Уведомления**:
   - Отправка оповещений при успешной/неуспешной замене сертификатов
   - Оповещения о скором истечении срока действия сертификатов

3. **Ротация ключей**:
   - Автоматическая генерация новых ключей при необходимости
   - Поддержка алгоритмов ECDSA и RSA

4. **Поддержка HSTS**:
   - Автоматическая настройка заголовков HSTS для веб-серверов

5. **Аварийный откат**:
   - Сохранение резервных копий старых сертификатов
   - Возможность быстрого отката при проблемах

Эта роль предоставляет гибкое решение для централизованного управления SSL/TLS сертификатами на всех серверах инфраструктуры с поддержкой основных веб- и почтовых сервисов.