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




Мы можем значительно улучшить предложенную ранее роль, сделав её более надёжной, гибкой и соответствующей лучшим практикам Ansible. Ниже перечислены ключевые направления улучшений с примерами реализации.

1. Идемпотентность и проверка существующих сертификатов

Сейчас роль всегда копирует новые сертификаты и создаёт бэкапы, даже если файлы не изменились. Это приводит к лишним бэкапам и перезапуску Apache.
Улучшение: сравнивать контрольные суммы или даты изменения новых и старых сертификатов; выполнять замену только при различиях.

```yaml
- name: Получить checksum текущего сертификата
  stat:
    path: "{{ apache_cert_file }}"
  register: old_cert_stat

- name: Вычислить checksum нового сертификата (локально)
  local_action: stat path="files/{{ new_cert_file }}"
  register: new_cert_local_stat
  run_once: yes

- name: Загрузить новый сертификат, если отличается
  copy:
    src: "{{ new_cert_file }}"
    dest: "{{ apache_cert_file }}"
    mode: '0644'
  when: new_cert_local_stat.stat.checksum != old_cert_stat.stat.checksum
  register: cert_updated
```

2. Бэкапы с ротацией и сжатием

Вместо простого копирования файлов с датой можно создать архив всех сертификатов и настроить ротацию (удаление старых бэкапов).

```yaml
- name: Создать временную директорию для бэкапа
  tempfile:
    state: directory
    suffix: apachecerts
  register: backup_temp

- name: Скопировать текущие сертификаты во временную папку
  copy:
    src: "{{ item }}"
    dest: "{{ backup_temp.path }}/"
    remote_src: yes
  loop:
    - "{{ apache_cert_file }}"
    - "{{ apache_key_file }}"
    - "{{ apache_chain_file }}"
  when: item is file

- name: Создать архив с меткой времени
  archive:
    path: "{{ backup_temp.path }}/*"
    dest: "{{ apache_certs_backup_dir }}/certs_{{ backup_timestamp }}.tar.gz"
    format: gz
    remove: yes

- name: Удалить старые бэкапы (оставить последние N)
  find:
    paths: "{{ apache_certs_backup_dir }}"
    patterns: "certs_*.tar.gz"
  register: old_backups

- name: Очистить старые бэкапы
  file:
    path: "{{ item.path }}"
    state: absent
  loop: "{{ old_backups.files | sort(attribute='mtime') | reverse | list | slice(keep_last_backups, old_backups.files | length) }}"
  vars:
    keep_last_backups: 5   # хранить последние 5 копий
```

3. Проверка валидности новых сертификатов до замены

Прежде чем положить новые сертификаты на целевой сервер, можно проверить их локально с помощью OpenSSL.

```yaml
- name: Проверить новый сертификат (локально)
  local_action: command openssl x509 -in files/{{ new_cert_file }} -text -noout
  changed_when: false
  ignore_errors: yes
  register: cert_check

- name: Остановить выполнение, если сертификат некорректен
  fail:
    msg: "Новый сертификат {{ new_cert_file }} не является валидным X.509"
  when: cert_check.rc != 0
```

4. Использование block и rescue для атомарности и отката

Если после замены проверка конфигурации Apache не проходит, нужно автоматически восстановить предыдущие сертификаты из бэкапа.

```yaml
- block:
    - name: Загрузить новые сертификаты
      copy: ...   # замена

    - name: Проверить конфигурацию Apache
      command: ...

    - name: Перезапустить Apache
      service: ...
  rescue:
    - name: Восстановить сертификаты из последнего бэкапа
      unarchive:
        src: "{{ latest_backup }}"
        dest: "{{ cert_dir_parent }}"
        remote_src: yes
      ...
    - name: Перезапустить Apache со старыми сертификатами
      service: ...
    - fail: msg="Не удалось заменить сертификаты, выполнена автоматическая откатка"
```

5. Гибкость в указании путей и имён файлов

Использовать переменные с разумными значениями по умолчанию, но позволять переопределять их для каждого хоста или группы. Хорошо бы также поддерживать ситуацию, когда chain-файл не нужен.

```yaml
apache_cert_src: "{{ new_cert_file | default('server.crt') }}"
apache_key_src: "{{ new_key_file | default('server.key') }}"
apache_chain_src: "{{ new_chain_file | default(omit) }}"
```

6. Работа с SELinux (особенно для Oracle Linux)

На системах с SELinux файлы сертификатов должны иметь правильный контекст. Добавим задачу для восстановления контекста.

```yaml
- name: Восстановить контекст SELinux для файлов сертификатов
  command: restorecon -Rv {{ cert_dir }} {{ key_dir }}
  when: ansible_selinux.status == "enabled"
```

7. Использование Handlers для перезапуска с проверкой

Handlers позволяют перезапускать Apache только в случае реальных изменений, но при этом важно проверить конфигурацию до перезапуска.

```yaml
# tasks/main.yml
- name: Загрузить новые сертификаты
  copy: ...
  notify: перезапустить apache с проверкой

# handlers/main.yml
- name: перезапустить apache с проверкой
  block:
    - name: Проверить конфигурацию
      command: ...
    - name: Перезапустить Apache
      service: ...
  rescue:
    - fail: msg="Конфигурация Apache неверна, перезапуск отменён"
```

8. Тегирование задач

Добавить теги, чтобы можно было выполнять только отдельные этапы: бэкап, загрузка сертификатов, проверка.

```yaml
- name: Создать бэкап
  ...
  tags: [backup, always]

- name: Загрузить сертификаты
  ...
  tags: [deploy, certs]

- name: Проверить конфигурацию
  ...
  tags: [validate]
```

9. Поддержка разных форматов (PEM, PKCS#12)

Можно добавить переменную, определяющую тип сертификата, и конвертировать при необходимости (например, с помощью openssl).

10. Документирование роли

Обязательно добавить README с описанием переменных, зависимостей, примеров.

---

Пример обновлённой задачи (основной блок с улучшениями)

```yaml
- name: Блок замены сертификатов
  block:
    - name: Проверить новые сертификаты локально
      local_action: command openssl x509 -in "files/{{ apache_cert_src }}" -text -noout
      changed_when: false
      register: local_cert_check
      ignore_errors: yes

    - name: Ошибка, если локальный сертификат невалиден
      fail:
        msg: "Файл {{ apache_cert_src }} не является корректным сертификатом X.509"
      when: local_cert_check.rc != 0

    - name: Получить информацию о текущих сертификатах
      stat:
        path: "{{ item }}"
      loop:
        - "{{ apache_cert_file }}"
        - "{{ apache_key_file }}"
        - "{{ apache_chain_file }}"
      register: current_files_stat

    - name: Создать бэкап, если хотя бы один файл изменился
      when: current_files_stat.results | map(attribute='stat.checksum') | list != new_checksums
      block:
        - name: Создать временную папку
          tempfile:
            state: directory
          register: tmpdir

        - name: Скопировать текущие файлы
          copy:
            src: "{{ item.stat.path }}"
            dest: "{{ tmpdir.path }}/"
            remote_src: yes
          loop: "{{ current_files_stat.results }}"
          when: item.stat.exists

        - name: Создать архив с датой
          archive:
            path: "{{ tmpdir.path }}/*"
            dest: "{{ backup_dir }}/certs_{{ ansible_date_time.epoch }}.tar.gz"
            format: gz
            remove: yes

    - name: Загрузить новые сертификаты
      copy:
        src: "{{ item.src }}"
        dest: "{{ item.dest }}"
        mode: "{{ item.mode }}"
        owner: root
        group: root
      loop:
        - { src: "{{ apache_cert_src }}", dest: "{{ apache_cert_file }}", mode: '0644' }
        - { src: "{{ apache_key_src }}", dest: "{{ apache_key_file }}", mode: '0640' }
        - { src: "{{ apache_chain_src | default(omit) }}", dest: "{{ apache_chain_file | default(omit) }}", mode: '0644' }
      when: item.src is defined and item.dest is defined
      notify: проверить и перезапустить apache

  rescue:
    - name: Восстановить сертификаты из последнего бэкапа
      unarchive:
        src: "{{ lookup('fileglob', backup_dir ~ '/certs_*.tar.gz') | sort | last }}"
        dest: "{{ (apache_cert_file | dirname) if ansible_os_family == 'Debian' else '/etc/pki/tls' }}"
        remote_src: yes
      when: backup_dir is defined

    - name: Перезапустить Apache со старыми сертификатами
      service:
        name: "{{ apache_service }}"
        state: restarted

    - fail: msg="Не удалось заменить сертификаты. Выполнен откат."
```

Эти улучшения сделают роль надёжной, безопасной и удобной в использовании в реальных условиях.