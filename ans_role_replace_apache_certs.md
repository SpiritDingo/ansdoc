Ниже представлена структура и примерное содержимое роли Ansible, которая выполняет замену SSL-сертификатов для Apache на Oracle Linux (RHEL-подобные) и Ubuntu, создавая резервную копию старых сертификатов с меткой времени.

Структура роли

```
replace_apache_certs/
├── tasks
│   └── main.yml
├── handlers
│   └── main.yml
├── vars
│   └── main.yml
├── defaults
│   └── main.yml
└── files
    └── (новые сертификаты, если они хранятся в роли)
```

Переменные (defaults/main.yml)

Здесь можно задать пути по умолчанию, которые будут переопределяться при необходимости.

```yaml
---
# Директория для резервных копий
apache_certs_backup_dir: /backup/apache_certs

# Пути к текущим сертификатам (зависят от ОС, будут переопределены в vars/)
apache_cert_file: /etc/ssl/certs/server.crt
apache_key_file: /etc/ssl/private/server.key
apache_chain_file: /etc/ssl/certs/chain.crt   # если есть промежуточные

# Имена новых сертификатов (ожидаются в files/ или переменных)
new_cert_file: new_server.crt
new_key_file: new_server.key
new_chain_file: new_chain.crt

# Флаг перезапуска Apache
restart_apache: yes
```

Переменные для разных ОС (vars/main.yml)

Используем ansible_os_family для выбора правильных путей.

```yaml
---
# Для Debian/Ubuntu
apache_cert_file: "/etc/ssl/certs/{{ apache_cert_name | default('server.crt') }}"
apache_key_file: "/etc/ssl/private/{{ apache_key_name | default('server.key') }}"
apache_chain_file: "/etc/ssl/certs/{{ apache_chain_name | default('chain.crt') }}"
apache_service: apache2

# Для RedHat/Oracle Linux
apache_cert_file: "/etc/pki/tls/certs/{{ apache_cert_name | default('server.crt') }}"
apache_key_file: "/etc/pki/tls/private/{{ apache_key_name | default('server.key') }}"
apache_chain_file: "/etc/pki/tls/certs/{{ apache_chain_name | default('chain.crt') }}"
apache_service: httpd
```

Задачи (tasks/main.yml)

```yaml
---
- name: Определить семейство ОС
  set_fact:
    apache_service: "{{ 'apache2' if ansible_os_family == 'Debian' else 'httpd' }}"
    cert_dir: "{{ '/etc/ssl/certs' if ansible_os_family == 'Debian' else '/etc/pki/tls/certs' }}"
    key_dir: "{{ '/etc/ssl/private' if ansible_os_family == 'Debian' else '/etc/pki/tls/private' }}"

- name: Создать директорию для резервных копий
  file:
    path: "{{ apache_certs_backup_dir }}"
    state: directory
    mode: '0750'

- name: Получить текущую дату и время для имени бэкапа
  set_fact:
    backup_timestamp: "{{ ansible_date_time.date }}_{{ ansible_date_time.hour }}{{ ansible_date_time.minute }}{{ ansible_date_time.second }}"

- name: Создать резервную копию текущих сертификатов
  copy:
    src: "{{ item.src }}"
    dest: "{{ apache_certs_backup_dir }}/{{ item.name }}_{{ backup_timestamp }}"
    remote_src: yes
    mode: preserve
  loop:
    - { src: "{{ apache_cert_file }}", name: "cert" }
    - { src: "{{ apache_key_file }}", name: "key" }
    - { src: "{{ apache_chain_file }}", name: "chain" }
  when: item.src is file
  ignore_errors: yes   # если какого-то файла нет – продолжаем

- name: Скопировать новые сертификаты на сервер
  copy:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    mode: "{{ item.mode }}"
    owner: root
    group: root
  loop:
    - { src: "{{ new_cert_file }}", dest: "{{ apache_cert_file }}", mode: '0644' }
    - { src: "{{ new_key_file }}", dest: "{{ apache_key_file }}", mode: '0640' }
    - { src: "{{ new_chain_file }}", dest: "{{ apache_chain_file }}", mode: '0644' }
  when: item.src is not none   # можно проверять существование файла в files/ или передавать содержимое

- name: Проверить конфигурацию Apache
  command: "{{ 'apache2ctl configtest' if ansible_os_family == 'Debian' else 'httpd -t' }}"
  register: apache_config_test
  changed_when: false

- name: Перезапустить Apache, если конфигурация верна и требуется
  service:
    name: "{{ apache_service }}"
    state: restarted
  when:
    - restart_apache | bool
    - apache_config_test.rc == 0
  notify: Проверить Apache после перезапуска  # можно добавить handler для проверки
```

Handlers (handlers/main.yml)

```yaml
---
- name: Проверить Apache после перезапуска
  wait_for:
    port: 443
    host: "{{ ansible_default_ipv4.address }}"
    delay: 5
    timeout: 30
  listen: "restart apache"
  ignore_errors: yes
```

Использование роли

Пример playbook:

```yaml
- hosts: apache_servers
  become: yes
  vars:
    # Указываем имена новых сертификатов (лежат в files/ роли)
    new_cert_file: wildcard_example_com.crt
    new_key_file: wildcard_example_com.key
    new_chain_file: intermediate.crt
    # При необходимости можно задать конкретные имена целевых файлов
    apache_cert_name: server.crt
    apache_key_name: server.key
    apache_chain_name: chain.crt
  roles:
    - replace_apache_certs
```

Примечания

· Новые сертификаты можно хранить в поддиректории files/ роли или передавать их содержимое через переменные (например, lookup('file', ...)). В примере используется copy с src из files/ (по умолчанию Ansible ищет там).
· Для ключей с ограниченным доступом (0600 или 0640) убедитесь, что владелец – root, группа – корневая или ssl-cert (на Ubuntu).
· Используйте ansible_date_time для генерации временной метки. Для этого факт должен быть включен (обычно включён по умолчанию, иначе можно добавить gather_facts: yes).
· В зависимости от настроек SELinux на Oracle Linux могут потребоваться дополнительные контексты. Если SELinux включён, проверьте типы контекстов для файлов сертификатов.
· Для отказоустойчивости можно добавить проверку доступности HTTPS после замены.

Эта роль гибко настраивается под конкретные пути и имена файлов, поддерживает оба семейства ОС и автоматически создаёт бэкапы с датой.