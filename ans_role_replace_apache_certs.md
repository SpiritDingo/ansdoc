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



Вот полная и улучшенная роль Ansible для замены SSL-сертификатов Apache на Oracle Linux и Ubuntu. Она включает автоматическое создание резервных копий с датой, проверку новых сертификатов, идемпотентность, поддержку SELinux и автоматический откат при ошибках.

Структура роли

```
replace_apache_certs/
├── tasks/
│   └── main.yml
├── handlers/
│   └── main.yml
├── defaults/
│   └── main.yml
├── vars/
│   └── main.yml
└── files/
    (сюда поместите новые сертификаты: server.crt, server.key, chain.crt при необходимости)
```

Файлы роли

defaults/main.yml

```yaml
---
# Директория для резервных копий
backup_dir: /backup/apache_certs

# Сколько последних бэкапов хранить
keep_backups: 5

# Перезапускать Apache после замены?
restart_apache: true

# Имена исходных файлов новых сертификатов (ожидаются в files/ роли)
apache_cert_src: "server.crt"
apache_key_src: "server.key"
apache_chain_src: ""   # если пусто, chain не используется

# Имена целевых файлов (без пути)
apache_cert_name: "server.crt"
apache_key_name: "server.key"
apache_chain_name: "chain.crt"
```

vars/main.yml

```yaml
---
# Переменные определяются динамически в tasks в зависимости от ОС
# Файл оставлен для совместимости
```

tasks/main.yml

```yaml
---
- name: Определить переменные в зависимости от ОС
  set_fact:
    apache_service: "{{ 'apache2' if ansible_os_family == 'Debian' else 'httpd' }}"
    apache_config_test_command: "{{ 'apache2ctl configtest' if ansible_os_family == 'Debian' else 'httpd -t' }}"
    apache_cert_dir: "{{ '/etc/ssl/certs' if ansible_os_family == 'Debian' else '/etc/pki/tls/certs' }}"
    apache_key_dir: "{{ '/etc/ssl/private' if ansible_os_family == 'Debian' else '/etc/pki/tls/private' }}"
    apache_chain_dir: "{{ '/etc/ssl/certs' if ansible_os_family == 'Debian' else '/etc/pki/tls/certs' }}"
    apache_key_group: "{{ 'ssl-cert' if ansible_os_family == 'Debian' else 'root' }}"

- name: Составить полные пути к целевым файлам
  set_fact:
    apache_cert_file: "{{ apache_cert_dir }}/{{ apache_cert_name }}"
    apache_key_file: "{{ apache_key_dir }}/{{ apache_key_name }}"
    apache_chain_file: "{{ apache_chain_dir }}/{{ apache_chain_name if apache_chain_name else '' }}"

- name: Собрать факты, если ещё не собраны (нужна дата)
  setup:
    gather_subset: "!all,min"
  when: ansible_date_time is not defined

- name: Создать директорию для бэкапов
  file:
    path: "{{ backup_dir }}"
    state: directory
    mode: '0750'
  tags: [backup, always]

- name: Проверить локальные файлы новых сертификатов
  local_action: stat path="files/{{ item }}"
  register: local_files_stat
  loop:
    - "{{ apache_cert_src }}"
    - "{{ apache_key_src }}"
  when: item != ''
  run_once: true
  tags: [always]

- name: Проверить валидность нового сертификата (локально)
  local_action: command openssl x509 -in "files/{{ apache_cert_src }}" -text -noout
  changed_when: false
  register: cert_validation
  ignore_errors: yes
  when: apache_cert_src != ''
  run_once: true
  tags: [validate]

- name: Остановить выполнение, если сертификат невалиден
  fail:
    msg: "Файл {{ apache_cert_src }} не является корректным сертификатом X.509"
  when: cert_validation is failed
  tags: [validate]

- name: Проверить наличие локального ключа
  local_action: stat path="files/{{ apache_key_src }}"
  register: key_stat
  when: apache_key_src != ''
  run_once: true

- name: Остановить выполнение, если ключ не найден
  fail:
    msg: "Ключ {{ apache_key_src }} не найден в files/"
  when: not key_stat.stat.exists
  tags: [validate]

# Если указан chain, проверим его наличие и валидность (опционально)
- name: Проверить локальный chain файл (если указан)
  local_action: stat path="files/{{ apache_chain_src }}"
  register: chain_stat
  when: apache_chain_src != ''
  run_once: true

- name: Проверить валидность chain (если есть)
  local_action: command openssl x509 -in "files/{{ apache_chain_src }}" -text -noout
  changed_when: false
  register: chain_validation
  ignore_errors: yes
  when: apache_chain_src != ''
  run_once: true

- name: Предупреждение, если chain невалиден
  debug:
    msg: "Внимание: chain файл {{ apache_chain_src }} не является валидным сертификатом, но продолжение возможно."
  when: chain_validation is failed and apache_chain_src != ''

# Получить информацию о текущих файлах на удалённом хосте
- name: Получить статус текущих сертификатов
  stat:
    path: "{{ item }}"
  loop:
    - "{{ apache_cert_file }}"
    - "{{ apache_key_file }}"
    - "{{ apache_chain_file if apache_chain_file != '' else omit }}"
  register: remote_files_stat
  tags: [always]

- name: Вычислить контрольные суммы новых файлов (локально)
  set_fact:
    new_cert_checksum: "{{ local_files_stat.results[0].stat.checksum | default('') }}"
    new_key_checksum: "{{ local_files_stat.results[1].stat.checksum | default('') }}"
    new_chain_checksum: "{{ chain_stat.stat.checksum | default('') if apache_chain_src != '' else '' }}"
  run_once: true
  tags: [always]

- name: Определить, нужно ли обновление (если хотя бы один файл отличается)
  set_fact:
    need_update: >-
      {{ (remote_files_stat.results[0].stat.checksum | default('') != new_cert_checksum) or
         (remote_files_stat.results[1].stat.checksum | default('') != new_key_checksum) or
         (apache_chain_src != '' and (remote_files_stat.results[2].stat.checksum | default('') != new_chain_checksum)) }}
  tags: [always]

- name: Блок замены сертификатов (если требуется обновление)
  when: need_update
  block:
    - name: Создать резервную копию текущих сертификатов
      block:
        - name: Создать временную директорию для бэкапа
          tempfile:
            state: directory
            suffix: apache_certs_backup
          register: backup_temp

        - name: Скопировать текущие файлы во временную папку
          copy:
            src: "{{ item.stat.path }}"
            dest: "{{ backup_temp.path }}/"
            remote_src: yes
          loop: "{{ remote_files_stat.results }}"
          when: item.stat is defined and item.stat.exists

        - name: Создать архив с меткой времени
          archive:
            path: "{{ backup_temp.path }}/*"
            dest: "{{ backup_dir }}/certs_{{ ansible_date_time.epoch }}.tar.gz"
            format: gz
            remove: yes
      rescue:
        - name: Очистить временную папку в случае ошибки
          file:
            path: "{{ backup_temp.path }}"
            state: absent
          when: backup_temp.path is defined
        - fail: msg="Не удалось создать резервную копию. Операция прервана."
      tags: [backup]

    - name: Загрузить новые сертификаты на сервер
      copy:
        src: "{{ item.src }}"
        dest: "{{ item.dest }}"
        mode: "{{ item.mode }}"
        owner: root
        group: "{{ item.group }}"
        backup: no
      loop:
        - { src: "{{ apache_cert_src }}", dest: "{{ apache_cert_file }}", mode: '0644', group: 'root' }
        - { src: "{{ apache_key_src }}", dest: "{{ apache_key_file }}", mode: '0640', group: "{{ apache_key_group }}" }
        - { src: "{{ apache_chain_src }}", dest: "{{ apache_chain_file }}", mode: '0644', group: 'root' }
      when: item.src != '' and item.dest != ''
      notify: перезапустить apache
      tags: [deploy]

    - name: Восстановить контекст SELinux (если включен)
      command: restorecon -Rv {{ apache_cert_dir }} {{ apache_key_dir }}
      when: ansible_selinux is defined and ansible_selinux.status == "enabled"
      changed_when: false
      tags: [deploy, selinux]

    - name: Проверить конфигурацию Apache
      command: "{{ apache_config_test_command }}"
      register: config_test
      changed_when: false
      tags: [deploy, validate]

    - name: Перезапустить Apache (если конфиг верен и требуется)
      service:
        name: "{{ apache_service }}"
        state: restarted
      when: restart_apache | bool and config_test.rc == 0
      tags: [deploy]

    - name: Ошибка, если конфиг неверен
      fail:
        msg: "Конфигурация Apache неверна. Запущен откат."
      when: config_test.rc != 0
      tags: [deploy, validate]

  rescue:
    - name: Восстановить сертификаты из последнего бэкапа в случае ошибки
      block:
        - name: Найти последний архив бэкапа
          find:
            paths: "{{ backup_dir }}"
            patterns: "certs_*.tar.gz"
          register: backup_files

        - name: Взять самый свежий файл
          set_fact:
            latest_backup: "{{ (backup_files.files | sort(attribute='mtime') | last).path }}"
          when: backup_files.files | length > 0

        - name: Распаковать последний бэкап
          unarchive:
            src: "{{ latest_backup }}"
            dest: "{{ apache_cert_dir | dirname if ansible_os_family == 'Debian' else '/etc/pki/tls' }}"
            remote_src: yes
          when: latest_backup is defined

        - name: Восстановить контекст SELinux
          command: restorecon -Rv {{ apache_cert_dir }} {{ apache_key_dir }}
          when: ansible_selinux is defined and ansible_selinux.status == "enabled"
          changed_when: false

        - name: Перезапустить Apache со старыми сертификатами
          service:
            name: "{{ apache_service }}"
            state: restarted
          when: latest_backup is defined
      rescue:
        - debug:
            msg: "Критическая ошибка при восстановлении. Проверьте вручную."

    - fail:
        msg: "Ошибка при замене сертификатов. Выполнена попытка отката. Проверьте логи."

  always:
    - name: Очистить ротацию старых бэкапов (оставить только последние N)
      block:
        - name: Найти все архивы бэкапов
          find:
            paths: "{{ backup_dir }}"
            patterns: "certs_*.tar.gz"
          register: all_backups

        - name: Отсортировать по времени и удалить старые
          file:
            path: "{{ item.path }}"
            state: absent
          loop: "{{ all_backups.files | sort(attribute='mtime') | reverse | list[keep_backups:] }}"
          when: all_backups.files | length > keep_backups
      tags: [backup, cleanup]

- name: Сообщить, что обновление не требуется
  debug:
    msg: "Сертификаты уже актуальны, обновление не требуется."
  when: not need_update
  tags: [always]
```

handlers/main.yml

```yaml
---
- name: перезапустить apache
  service:
    name: "{{ apache_service }}"
    state: restarted
  when: restart_apache | bool
```

Использование

1. Поместите новые сертификаты в директорию files/ роли (например, files/server.crt, files/server.key, files/chain.crt).
2. В playbook укажите необходимые переменные (если нужно переопределить имена файлов или пути).
3. Запустите плейбук.

Пример playbook:

```yaml
- hosts: apache_servers
  become: yes
  vars:
    apache_cert_src: "wildcard.example.com.crt"
    apache_key_src: "wildcard.example.com.key"
    apache_chain_src: "ca-bundle.crt"
    backup_dir: /opt/backups/apache_certs
    keep_backups: 10
  roles:
    - replace_apache_certs
```

Особенности

· Поддержка двух ОС: автоматически определяет пути и команды для Ubuntu (Debian) и Oracle Linux (RedHat).
· Проверка новых сертификатов локально с помощью openssl до начала замены.
· Резервное копирование всех текущих файлов в архив с временной меткой (epoch).
· Ротация бэкапов – оставляет только последние N копий.
· Идемпотентность – сравнивает контрольные суммы и выполняет замену только при изменениях.
· SELinux – восстанавливает контекст после замены (если включен).
· Автоматический откат – при ошибке проверки конфигурации Apache восстанавливает предыдущие сертификаты из последнего бэкапа и перезапускает сервис.
· Теги – позволяют запускать только отдельные этапы: backup, deploy, validate, cleanup.







Ошибка возникает из-за того, что задача проверки валидности сертификата запускается даже если файл физически отсутствует в каталоге files/. Хотя мы игнорируем ошибку (ignore_errors: yes), последующая задача fail срабатывает, потому что регистр cert_validation помечен как failed. Нужно проверять существование файла перед попыткой его валидации и уточнить сообщение об ошибке.

Ниже приведён исправленный блок задач, относящихся к проверке локальных файлов и валидации сертификатов (заменяет соответствующие части в tasks/main.yml).

Исправленный фрагмент tasks/main.yml

```yaml
# ... (предыдущие задачи до проверки локальных файлов)

- name: Проверить локальные файлы новых сертификатов
  local_action: stat path="files/{{ item }}"
  register: local_files_stat
  loop:
    - "{{ apache_cert_src }}"
    - "{{ apache_key_src }}"
  when: item != ''
  run_once: true
  tags: [always]

- name: Проверить существование файла сертификата
  fail:
    msg: "Файл сертификата {{ apache_cert_src }} не найден в директории files/ роли."
  when: not local_files_stat.results[0].stat.exists
  run_once: true
  tags: [validate]

- name: Проверить валидность нового сертификата (локально)
  local_action: command openssl x509 -in "files/{{ apache_cert_src }}" -text -noout
  changed_when: false
  register: cert_validation
  ignore_errors: yes
  when: local_files_stat.results[0].stat.exists   # файл уже точно есть, но проверка на всякий случай
  run_once: true
  tags: [validate]

- name: Остановить выполнение, если сертификат невалиден
  fail:
    msg: "Файл {{ apache_cert_src }} присутствует, но не является корректным сертификатом X.509 (ошибка openssl)."
  when: cert_validation is failed
  tags: [validate]

# Аналогично для ключа (можно не проверять валидность openssl, только существование)
- name: Проверить существование файла ключа
  fail:
    msg: "Файл ключа {{ apache_key_src }} не найден в директории files/ роли."
  when: not local_files_stat.results[1].stat.exists
  run_once: true
  tags: [validate]

# Для chain (если указан)
- name: Проверить локальный chain файл (если указан)
  local_action: stat path="files/{{ apache_chain_src }}"
  register: chain_stat
  when: apache_chain_src != ''
  run_once: true

- name: Проверить существование chain файла (если указан)
  fail:
    msg: "Файл chain {{ apache_chain_src }} указан, но не найден в files/."
  when: apache_chain_src != '' and not chain_stat.stat.exists
  run_once: true
  tags: [validate]

- name: Проверить валидность chain (если есть)
  local_action: command openssl x509 -in "files/{{ apache_chain_src }}" -text -noout
  changed_when: false
  register: chain_validation
  ignore_errors: yes
  when: apache_chain_src != '' and chain_stat.stat.exists
  run_once: true

- name: Предупреждение, если chain невалиден
  debug:
    msg: "Внимание: chain файл {{ apache_chain_src }} не является валидным сертификатом, но продолжение возможно."
  when: chain_validation is failed and apache_chain_src != ''
  tags: [validate]

# ... (остальные задачи)
```

Что изменилось

1. Проверка существования файла сертификата до его валидации. Если файла нет – выполнение прерывается с понятным сообщением.
2. Проверка валидности теперь выполняется только при гарантированном наличии файла (условие when). Это предотвращает ложные срабатывания openssl.
3. Сообщение об ошибке уточнено: теперь различается ситуация "файл отсутствует" и "файл есть, но невалиден".
4. Аналогичные проверки добавлены для ключа (только существование, так как ключ не обязан быть в формате PEM для проверки через openssl) и для chain-файла.

После этих исправлений роль будет корректно обрабатывать отсутствие файлов и не будет пытаться проверить несуществующий сертификат.



Мы добавили возможность указать произвольные пути к целевым файлам сертификатов на сервере. Теперь пользователь может задать полные пути через переменные:

· apache_cert_file
· apache_key_file
· apache_chain_file

Если эти переменные не заданы, пути формируются автоматически на основе ОС и переменных apache_cert_name, apache_key_name, apache_chain_name (как и раньше). Это даёт гибкость при нестандартных расположениях сертификатов.

Ниже приведены полные обновлённые файлы роли с учётом этого улучшения и предыдущих исправлений (проверка существования локальных файлов, валидация, идемпотентность, откат и т.д.).

---

Структура роли

```
replace_apache_certs/
├── tasks/
│   └── main.yml
├── handlers/
│   └── main.yml
├── defaults/
│   └── main.yml
└── files/
    (сюда поместите новые сертификаты)
```

defaults/main.yml

```yaml
---
# Директория для резервных копий
backup_dir: /backup/apache_certs

# Сколько последних бэкапов хранить
keep_backups: 5

# Перезапускать Apache после замены?
restart_apache: true

# Имена исходных файлов новых сертификатов (ожидаются в files/ роли)
apache_cert_src: "server.crt"
apache_key_src: "server.key"
apache_chain_src: ""   # если пусто, chain не используется

# Имена целевых файлов (без пути). Используются, если не заданы полные пути ниже.
apache_cert_name: "server.crt"
apache_key_name: "server.key"
apache_chain_name: "chain.crt"

# Полные пути к целевым файлам на сервере.
# Если эти переменные заданы, они имеют приоритет над составными из каталогов+имена.
# apache_cert_file: /custom/path/server.crt
# apache_key_file: /custom/path/server.key
# apache_chain_file: /custom/path/chain.crt
```

handlers/main.yml

```yaml
---
- name: перезапустить apache
  service:
    name: "{{ apache_service }}"
    state: restarted
  when: restart_apache | bool
```

tasks/main.yml

```yaml
---
- name: Определить переменные в зависимости от ОС
  set_fact:
    apache_service: "{{ 'apache2' if ansible_os_family == 'Debian' else 'httpd' }}"
    apache_config_test_command: "{{ 'apache2ctl configtest' if ansible_os_family == 'Debian' else 'httpd -t' }}"
    apache_default_cert_dir: "{{ '/etc/ssl/certs' if ansible_os_family == 'Debian' else '/etc/pki/tls/certs' }}"
    apache_default_key_dir: "{{ '/etc/ssl/private' if ansible_os_family == 'Debian' else '/etc/pki/tls/private' }}"
    apache_key_group: "{{ 'ssl-cert' if ansible_os_family == 'Debian' else 'root' }}"

- name: Определить конечные пути к файлам сертификатов (приоритет у явно заданных)
  set_fact:
    apache_cert_file: "{{ apache_cert_file | default(apache_default_cert_dir ~ '/' ~ apache_cert_name) }}"
    apache_key_file: "{{ apache_key_file | default(apache_default_key_dir ~ '/' ~ apache_key_name) }}"
    apache_chain_file: "{{ apache_chain_file | default(apache_default_cert_dir ~ '/' ~ apache_chain_name) if apache_chain_name else '' }}"

- name: Собрать факты, если ещё не собраны (нужна дата)
  setup:
    gather_subset: "!all,min"
  when: ansible_date_time is not defined

- name: Создать директорию для бэкапов
  file:
    path: "{{ backup_dir }}"
    state: directory
    mode: '0750'
  tags: [backup, always]

- name: Проверить локальные файлы новых сертификатов
  local_action: stat path="files/{{ item }}"
  register: local_files_stat
  loop:
    - "{{ apache_cert_src }}"
    - "{{ apache_key_src }}"
  when: item != ''
  run_once: true
  tags: [always]

- name: Проверить существование файла сертификата
  fail:
    msg: "Файл сертификата {{ apache_cert_src }} не найден в директории files/ роли."
  when: not local_files_stat.results[0].stat.exists
  run_once: true
  tags: [validate]

- name: Проверить валидность нового сертификата (локально)
  local_action: command openssl x509 -in "files/{{ apache_cert_src }}" -text -noout
  changed_when: false
  register: cert_validation
  ignore_errors: yes
  when: local_files_stat.results[0].stat.exists
  run_once: true
  tags: [validate]

- name: Остановить выполнение, если сертификат невалиден
  fail:
    msg: "Файл {{ apache_cert_src }} присутствует, но не является корректным сертификатом X.509 (ошибка openssl)."
  when: cert_validation is failed
  tags: [validate]

- name: Проверить существование файла ключа
  fail:
    msg: "Файл ключа {{ apache_key_src }} не найден в директории files/ роли."
  when: not local_files_stat.results[1].stat.exists
  run_once: true
  tags: [validate]

- name: Проверить локальный chain файл (если указан)
  local_action: stat path="files/{{ apache_chain_src }}"
  register: chain_stat
  when: apache_chain_src != ''
  run_once: true

- name: Проверить существование chain файла (если указан)
  fail:
    msg: "Файл chain {{ apache_chain_src }} указан, но не найден в files/."
  when: apache_chain_src != '' and not chain_stat.stat.exists
  run_once: true
  tags: [validate]

- name: Проверить валидность chain (если есть)
  local_action: command openssl x509 -in "files/{{ apache_chain_src }}" -text -noout
  changed_when: false
  register: chain_validation
  ignore_errors: yes
  when: apache_chain_src != '' and chain_stat.stat.exists
  run_once: true

- name: Предупреждение, если chain невалиден
  debug:
    msg: "Внимание: chain файл {{ apache_chain_src }} не является валидным сертификатом, но продолжение возможно."
  when: chain_validation is failed and apache_chain_src != ''
  tags: [validate]

- name: Получить статус текущих сертификатов на сервере
  stat:
    path: "{{ item }}"
  loop:
    - "{{ apache_cert_file }}"
    - "{{ apache_key_file }}"
    - "{{ apache_chain_file if apache_chain_file != '' else omit }}"
  register: remote_files_stat
  tags: [always]

- name: Вычислить контрольные суммы новых файлов (локально)
  set_fact:
    new_cert_checksum: "{{ local_files_stat.results[0].stat.checksum | default('') }}"
    new_key_checksum: "{{ local_files_stat.results[1].stat.checksum | default('') }}"
    new_chain_checksum: "{{ chain_stat.stat.checksum | default('') if apache_chain_src != '' else '' }}"
  run_once: true
  tags: [always]

- name: Определить, нужно ли обновление (если хотя бы один файл отличается)
  set_fact:
    need_update: >-
      {{ (remote_files_stat.results[0].stat.checksum | default('') != new_cert_checksum) or
         (remote_files_stat.results[1].stat.checksum | default('') != new_key_checksum) or
         (apache_chain_src != '' and (remote_files_stat.results[2].stat.checksum | default('') != new_chain_checksum)) }}
  tags: [always]

- name: Блок замены сертификатов (если требуется обновление)
  when: need_update
  block:
    - name: Создать резервную копию текущих сертификатов
      block:
        - name: Создать временную директорию для бэкапа
          tempfile:
            state: directory
            suffix: apache_certs_backup
          register: backup_temp

        - name: Скопировать текущие файлы во временную папку
          copy:
            src: "{{ item.stat.path }}"
            dest: "{{ backup_temp.path }}/"
            remote_src: yes
          loop: "{{ remote_files_stat.results }}"
          when: item.stat is defined and item.stat.exists

        - name: Создать архив с меткой времени
          archive:
            path: "{{ backup_temp.path }}/*"
            dest: "{{ backup_dir }}/certs_{{ ansible_date_time.epoch }}.tar.gz"
            format: gz
            remove: yes
      rescue:
        - name: Очистить временную папку в случае ошибки
          file:
            path: "{{ backup_temp.path }}"
            state: absent
          when: backup_temp.path is defined
        - fail: msg="Не удалось создать резервную копию. Операция прервана."
      tags: [backup]

    - name: Загрузить новые сертификаты на сервер
      copy:
        src: "{{ item.src }}"
        dest: "{{ item.dest }}"
        mode: "{{ item.mode }}"
        owner: root
        group: "{{ item.group }}"
        backup: no
      loop:
        - { src: "{{ apache_cert_src }}", dest: "{{ apache_cert_file }}", mode: '0644', group: 'root' }
        - { src: "{{ apache_key_src }}", dest: "{{ apache_key_file }}", mode: '0640', group: "{{ apache_key_group }}" }
        - { src: "{{ apache_chain_src }}", dest: "{{ apache_chain_file }}", mode: '0644', group: 'root' }
      when: item.src != '' and item.dest != ''
      notify: перезапустить apache
      tags: [deploy]

    - name: Восстановить контекст SELinux (если включен)
      command: restorecon -Rv {{ apache_default_cert_dir }} {{ apache_default_key_dir }}
      when: ansible_selinux is defined and ansible_selinux.status == "enabled"
      changed_when: false
      tags: [deploy, selinux]

    - name: Проверить конфигурацию Apache
      command: "{{ apache_config_test_command }}"
      register: config_test
      changed_when: false
      tags: [deploy, validate]

    - name: Перезапустить Apache (если конфиг верен и требуется)
      service:
        name: "{{ apache_service }}"
        state: restarted
      when: restart_apache | bool and config_test.rc == 0
      tags: [deploy]

    - name: Ошибка, если конфиг неверен
      fail:
        msg: "Конфигурация Apache неверна. Запущен откат."
      when: config_test.rc != 0
      tags: [deploy, validate]

  rescue:
    - name: Восстановить сертификаты из последнего бэкапа в случае ошибки
      block:
        - name: Найти последний архив бэкапа
          find:
            paths: "{{ backup_dir }}"
            patterns: "certs_*.tar.gz"
          register: backup_files

        - name: Взять самый свежий файл
          set_fact:
            latest_backup: "{{ (backup_files.files | sort(attribute='mtime') | last).path }}"
          when: backup_files.files | length > 0

        - name: Распаковать последний бэкап (внимание: при нестандартных путях может потребоваться ручная проверка)
          unarchive:
            src: "{{ latest_backup }}"
            dest: "{{ apache_default_cert_dir | dirname if ansible_os_family == 'Debian' else '/etc/pki/tls' }}"
            remote_src: yes
          when: latest_backup is defined

        - name: Восстановить контекст SELinux
          command: restorecon -Rv {{ apache_default_cert_dir }} {{ apache_default_key_dir }}
          when: ansible_selinux is defined and ansible_selinux.status == "enabled"
          changed_when: false

        - name: Перезапустить Apache со старыми сертификатами
          service:
            name: "{{ apache_service }}"
            state: restarted
          when: latest_backup is defined
      rescue:
        - debug:
            msg: "Критическая ошибка при восстановлении. Проверьте вручную."

    - fail:
        msg: "Ошибка при замене сертификатов. Выполнена попытка отката. Проверьте логи."

  always:
    - name: Очистить ротацию старых бэкапов (оставить только последние N)
      block:
        - name: Найти все архивы бэкапов
          find:
            paths: "{{ backup_dir }}"
            patterns: "certs_*.tar.gz"
          register: all_backups

        - name: Отсортировать по времени и удалить старые
          file:
            path: "{{ item.path }}"
            state: absent
          loop: "{{ all_backups.files | sort(attribute='mtime') | reverse | list[keep_backups:] }}"
          when: all_backups.files | length > keep_backups
      tags: [backup, cleanup]

- name: Сообщить, что обновление не требуется
  debug:
    msg: "Сертификаты уже актуальны, обновление не требуется."
  when: not need_update
  tags: [always]
```

---

Как использовать с произвольными путями

Пример playbook, где сертификаты лежат в нестандартных местах:

```yaml
- hosts: apache_servers
  become: yes
  vars:
    apache_cert_src: "example.com.crt"
    apache_key_src: "example.com.key"
    apache_chain_src: "ca.crt"
    # Явно указываем полные пути на целевой системе
    apache_cert_file: /opt/apache/ssl/example.crt
    apache_key_file: /opt/apache/private/example.key
    apache_chain_file: /opt/apache/ssl/ca.crt
    backup_dir: /opt/backups/apache_certs
  roles:
    - replace_apache_certs
```

Если переменные apache_cert_file и т.д. не заданы, роль автоматически использует стандартные пути для Ubuntu или Oracle Linux, подставляя имена из apache_cert_name и т.п.

---

Важно

· При использовании нестандартных путей автоматический откат из резервной копии может не восстановить файлы в правильные места, так как архив не сохраняет полную структуру каталогов. В таком случае рекомендуется после выполнения вручную проверить состояние сертификатов или доработать роль под свои нужды.
· Для ключей задаётся группа apache_key_group (ssl-cert на Ubuntu, root на Oracle Linux), чтобы обеспечить корректный доступ.



Мы добавили поддержку произвольных локальных путей к исходным файлам сертификатов. Теперь можно указывать:

· просто имя файла (тогда он ищется в поддиректории files/ роли), например: apache_cert_src: "Y1CAPDN.crt"
· абсолютный путь на управляющей машине (начинается с /), например: apache_cert_src: "/tmp/Y1CAPDN.crt"

Это позволяет гибко задавать местоположение новых сертификатов без необходимости копировать их внутрь структуры роли.

Изменения в tasks/main.yml (ключевые блоки)

Ниже приведён обновлённый код с комментариями. Замените соответствующие части вашего файла tasks/main.yml.

1. Определение фактических локальных путей

Добавьте эту задачу сразу после определения переменных ОС (или в начале раздела проверок):

```yaml
- name: Определить полные локальные пути к исходным файлам
  set_fact:
    local_cert_path: "{{ apache_cert_src if apache_cert_src.startswith('/') else role_path + '/files/' + apache_cert_src }}"
    local_key_path: "{{ apache_key_src if apache_key_src.startswith('/') else role_path + '/files/' + apache_key_src }}"
    local_chain_path: "{{ apache_chain_src if apache_chain_src.startswith('/') else (role_path + '/files/' + apache_chain_src) if apache_chain_src else '' }}"
  run_once: true
  tags: [always]
```

2. Проверка локальных файлов (используем вычисленные пути)

```yaml
- name: Проверить локальные файлы новых сертификатов
  local_action: stat path="{{ item }}"
  register: local_files_stat
  loop:
    - "{{ local_cert_path }}"
    - "{{ local_key_path }}"
  when: item != ''
  run_once: true
  tags: [always]
```

3. Проверка существования сертификата

```yaml
- name: Проверить существование файла сертификата
  fail:
    msg: "Файл сертификата {{ apache_cert_src }} не найден (искали по пути: {{ local_cert_path }})."
  when: not local_files_stat.results[0].stat.exists
  run_once: true
  tags: [validate]
```

4. Проверка валидности (используем локальный путь)

```yaml
- name: Проверить валидность нового сертификата (локально)
  local_action: command openssl x509 -in "{{ local_cert_path }}" -text -noout
  changed_when: false
  register: cert_validation
  ignore_errors: yes
  when: local_files_stat.results[0].stat.exists
  run_once: true
  tags: [validate]
```

5. Проверка ключа

```yaml
- name: Проверить существование файла ключа
  fail:
    msg: "Файл ключа {{ apache_key_src }} не найден (искали по пути: {{ local_key_path }})."
  when: not local_files_stat.results[1].stat.exists
  run_once: true
  tags: [validate]
```

6. Проверка chain-файла (аналогично)

```yaml
- name: Проверить локальный chain файл (если указан)
  local_action: stat path="{{ local_chain_path }}"
  register: chain_stat
  when: apache_chain_src != ''
  run_once: true

- name: Проверить существование chain файла (если указан)
  fail:
    msg: "Файл chain {{ apache_chain_src }} указан, но не найден (искали по пути: {{ local_chain_path }})."
  when: apache_chain_src != '' and not chain_stat.stat.exists
  run_once: true
  tags: [validate]

- name: Проверить валидность chain (если есть)
  local_action: command openssl x509 -in "{{ local_chain_path }}" -text -noout
  changed_when: false
  register: chain_validation
  ignore_errors: yes
  when: apache_chain_src != '' and chain_stat.stat.exists
  run_once: true
```

7. Вычисление контрольных сумм (используем локальные пути)

```yaml
- name: Вычислить контрольные суммы новых файлов (локально)
  set_fact:
    new_cert_checksum: "{{ local_files_stat.results[0].stat.checksum | default('') }}"
    new_key_checksum: "{{ local_files_stat.results[1].stat.checksum | default('') }}"
    new_chain_checksum: "{{ chain_stat.stat.checksum | default('') if apache_chain_src != '' else '' }}"
  run_once: true
  tags: [always]
```

8. Загрузка новых сертификатов (используем локальные пути в src)

```yaml
- name: Загрузить новые сертификаты на сервер
  copy:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    mode: "{{ item.mode }}"
    owner: root
    group: "{{ item.group }}"
    backup: no
  loop:
    - { src: "{{ local_cert_path }}", dest: "{{ apache_cert_file }}", mode: '0644', group: 'root' }
    - { src: "{{ local_key_path }}", dest: "{{ apache_key_file }}", mode: '0640', group: "{{ apache_key_group }}" }
    - { src: "{{ local_chain_path }}", dest: "{{ apache_chain_file }}", mode: '0644', group: 'root' }
  when: item.src != '' and item.dest != ''
  notify: перезапустить apache
  tags: [deploy]
```

Пример использования с произвольным путём

```yaml
- hosts: apache_servers
  become: yes
  vars:
    apache_cert_src: "/home/user/certs/Y1CAPDN.crt"
    apache_key_src: "/home/user/certs/Y1CAPDN.key"
    apache_chain_src: ""   # если не нужен
    # можно также указать целевые пути, если они нестандартные
    apache_cert_file: /etc/ssl/certs/Y1CAPDN.crt
    apache_key_file: /etc/ssl/private/Y1CAPDN.key
  roles:
    - replace_apache_certs
```

Важные замечания

· Переменная role_path автоматически доступна внутри роли и указывает на её корневую директорию.
· Если файл не найден ни по абсолютному пути, ни в files/, выполнение прерывается с понятным сообщением, содержащим оба варианта.
· При использовании относительных путей (не абсолютных) роль по-прежнему ищет файлы в files/ (стандартное поведение Ansible для копирования файлов из роли). Благодаря явному указанию role_path мы сохраняем совместимость и прозрачность.

Теперь роль полностью поддерживает как локальные пути на control node (абсолютные), так и файлы внутри структуры роли (относительные).
