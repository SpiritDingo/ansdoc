Вот Ansible роль для добавления корневых сертификатов в хранилище Google Chrome для указанных пользователей:

Структура роли:

```
roles/add-chrome-ca-cert/
├── tasks
│   └── main.yml
├── defaults
│   └── main.yml
└── templates
    └── import-cert.sh.j2
```

Файлы роли:

1. defaults/main.yml:

```yaml
---
# Список пользователей для настройки
chrome_users:
  - username: "user1"
    full_name: "First User"
  - username: "user2"
    full_name: "Second User"

# Путь к сертификату
ca_cert_path: "/usr/local/share/ca-certificates/CisRootCA.crt"

# Имя сертификата в хранилище
ca_cert_name: "CisRootCA"
```

2. tasks/main.yml:

```yaml
---
- name: Установка libnss3-tools
  apt:
    name: libnss3-tools
    state: present
    update_cache: yes

- name: Создание скрипта импорта сертификата
  template:
    src: import-cert.sh.j2
    dest: /tmp/import-cert.sh
    mode: '0755'

- name: Добавление сертификата в системное хранилище
  copy:
    src: "{{ ca_cert_path }}"
    dest: /usr/local/share/ca-certificates/
    mode: '0644'
  when: ca_cert_path is defined

- name: Обновление системных сертификатов
  command: update-ca-certificates
  when: ca_cert_path is defined

- name: Импорт сертификата для каждого пользователя
  become: yes
  become_user: "{{ item.username }}"
  shell: |
    # Создаем директорию если не существует
    mkdir -p /home/{{ item.username }}/.pki/nssdb
    
    # Проверяем наличие БД, если нет - создаем
    if [ ! -f /home/{{ item.username }}/.pki/nssdb/cert9.db ]; then
      certutil -d sql:/home/{{ item.username }}/.pki/nssdb -N --empty-password
    fi
    
    # Добавляем сертификат
    certutil -d sql:/home/{{ item.username }}/.pki/nssdb -A -t "C,," \
      -n "{{ ca_cert_name }}" -i /usr/local/share/ca-certificates/{{ ca_cert_name }}.crt
  loop: "{{ chrome_users }}"
  when: ca_cert_path is defined

- name: Проверка импортированных сертификатов
  become: yes
  become_user: "{{ item.username }}"
  command: certutil -d sql:/home/{{ item.username }}/.pki/nssdb -L
  loop: "{{ chrome_users }}"
  register: cert_check
  changed_when: false

- name: Вывод информации о сертификатах
  debug:
    msg: "Сертификаты для {{ item.item.username }}:\n{{ item.stdout }}"
  loop: "{{ cert_check.results }}"
  loop_control:
    label: "{{ item.item.username }}"
```

3. templates/import-cert.sh.j2:

```bash
#!/bin/bash
# Скрипт для импорта сертификатов в хранилище NSS

CERT_NAME="{{ ca_cert_name }}"
CERT_FILE="{{ ca_cert_path }}"
USER="$1"

if [ -z "$USER" ]; then
    echo "Использование: $0 <username>"
    exit 1
fi

HOME_DIR="/home/$USER"
NSSDB="$HOME_DIR/.pki/nssdb"

# Создаем директорию если не существует
mkdir -p "$NSSDB"

# Проверяем наличие БД, если нет - создаем
if [ ! -f "$NSSDB/cert9.db" ]; then
    certutil -d "sql:$NSSDB" -N --empty-password
fi

# Добавляем сертификат
certutil -d "sql:$NSSDB" -A -t "C,," \
    -n "$CERT_NAME" -i "$CERT_FILE"

echo "Сертификат $CERT_NAME добавлен для пользователя $USER"
```

Пример playbook:

```yaml
---
- name: Добавление CA сертификатов в Chrome
  hosts: all
  become: yes
  
  vars:
    chrome_users:
      - username: "alice"
        full_name: "Alice Smith"
      - username: "bob"
        full_name: "Bob Johnson"
    
    ca_cert_path: "/path/to/your/CisRootCA.crt"
    ca_cert_name: "MyCompanyRootCA"
  
  roles:
    - add-chrome-ca-cert
```

Использование:

1. Создайте структуру директорий роли:

```bash
mkdir -p roles/add-chrome-ca-cert/{tasks,defaults,templates}
```

1. Поместите файлы в соответствующие директории
2. Создайте playbook или добавьте роль в существующий
3. Запустите:

```bash
ansible-playbook -i inventory.yml playbook.yml
```

Альтернативный подход с использованием модуля shell:

Если предпочитаете более компактное решение:

```yaml
---
- name: Установка зависимостей
  apt:
    name: libnss3-tools
    state: present

- name: Импорт сертификатов для пользователей
  vars:
    cert_path: "/usr/local/share/ca-certificates/CisRootCA.crt"
    cert_name: "CisRootCA"
  loop: "{{ ['user1', 'user2', 'user3'] }}"
  block:
    - name: Создание директории NSS
      file:
        path: "/home/{{ item }}/.pki/nssdb"
        state: directory
        owner: "{{ item }}"
        group: "{{ item }}"
        mode: '0755'
    
    - name: Инициализация БД сертификатов
      shell: |
        certutil -d sql:/home/{{ item }}/.pki/nssdb -N --empty-password
      args:
        creates: "/home/{{ item }}/.pki/nssdb/cert9.db"
    
    - name: Добавление сертификата
      shell: |
        certutil -d sql:/home/{{ item }}/.pki/nssdb -A \
          -t "C,," \
          -n "{{ cert_name }}" \
          -i "{{ cert_path }}"
```

Примечания:

1. Сертификат добавляется с доверием C (trusted CA), но без SSL/TLS (,) и без подписи кода (последняя запятая)
2. Для работы в Firefox используется то же хранилище NSS
3. Для применения изменений может потребоваться перезапуск Chrome
4. На старых системах может использоваться cert8.db вместо cert9.db
5. Убедитесь, что сертификат в формате PEM/DER и доступен на целевых хостах