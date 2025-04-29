# Ansible роль для проверки сроков действия SSL/TLS сертификатов

Эта роль проверяет сроки действия сертификатов на серверах, включая веб-серверы, службы и файловые сертификаты.

## Структура роли

```
certificate_checker/
├── defaults/
│   └── main.yml       # Значения переменных по умолчанию
├── tasks/
│   └── main.yml       # Основные задачи проверки
├── templates/
│   └── report.j2      # Шаблон отчета
└── vars/
    └── main.yml       # Переменные роли
```

## Основные задачи роли (tasks/main.yml)

```yaml
---
- name: Создать директорию для отчетов
  file:
    path: "{{ report_dir }}"
    state: directory
    mode: 0755

- name: Проверить сертификаты веб-серверов
  uri:
    url: "https://{{ item }}{{ cert_check_web_path }}"
    method: GET
    validate_certs: no
    timeout: "{{ cert_check_timeout }}"
    return_content: no
  register: web_certs
  ignore_errors: yes
  loop: "{{ cert_check_domains }}"
  loop_control:
    loop_var: item
    label: "Checking {{ item }}"

- name: Извлечь информацию о сертификатах веб-серверов
  command: >
    openssl s_client -showcerts -connect {{ item.item }}:443 -servername {{ item.item }} </dev/null 2>/dev/null |
    openssl x509 -noout -dates -subject -issuer
  register: web_cert_info
  loop: "{{ web_certs.results }}"
  when: item is not failed
  changed_when: false

- name: Найти сертификаты в файловой системе
  find:
    paths: "{{ cert_search_paths }}"
    patterns: "*.pem,*.crt,*.cer,*.key"
    recurse: yes
    use_regex: no
  register: found_cert_files

- name: Проверить сроки действия файловых сертификатов
  command: >
    openssl x509 -noout -dates -subject -issuer -in {{ item.path }}
  register: file_cert_info
  loop: "{{ found_cert_files.files }}"
  changed_when: false

- name: Проверить сертификаты служб (LDAP, SMTP и т.д.)
  command: >
    openssl s_client -connect {{ item.host }}:{{ item.port }} -showcerts </dev/null 2>/dev/null |
    openssl x509 -noout -dates -subject -issuer
  register: service_cert_info
  loop: "{{ service_checks }}"
  changed_when: false

- name: Сгенерировать отчет
  template:
    src: templates/report.j2
    dest: "{{ report_dir }}/certificate_report_{{ ansible_hostname }}_{{ ansible_date_time.date }}.html"
    mode: 0644
```

## Переменные по умолчанию (defaults/main.yml)

```yaml
---
# Директория для сохранения отчетов
report_dir: "/var/log/certificate_reports"

# Таймаут проверки сертификатов (сек)
cert_check_timeout: 10

# Домены для проверки
cert_check_domains:
  - "{{ ansible_hostname }}"
  - localhost

# Путь для проверки веб-серверов
cert_check_web_path: "/"

# Пути для поиска файлов сертификатов
cert_search_paths:
  - /etc/ssl
  - /etc/pki
  - /etc/letsencrypt
  - /etc/nginx
  - /etc/apache2
  - /etc/httpd

# Сервисы для проверки
service_checks:
  - { host: "localhost", port: 636, name: "LDAPS" }
  - { host: "localhost", port: 993, name: "IMAPS" }
  - { host: "localhost", port: 465, name: "SMTPS" }

# Критическое количество дней до истечения срока
critical_days: 15
warning_days: 30
```

## Шаблон отчета (templates/report.j2)

```jinja2
<!DOCTYPE html>
<html>
<head>
    <title>Отчет о сертификатах - {{ ansible_hostname }}</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        h1 { color: #333; }
        table { border-collapse: collapse; width: 100%; margin-bottom: 20px; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
        .critical { background-color: #ffcccc; }
        .warning { background-color: #fff3cd; }
        .ok { background-color: #d4edda; }
    </style>
</head>
<body>
    <h1>Отчет о проверке сертификатов</h1>
    <p><strong>Сервер:</strong> {{ ansible_hostname }}</p>
    <p><strong>Дата проверки:</strong> {{ ansible_date_time.date }}</p>

    <h2>Веб-сертификаты</h2>
    <table>
        <tr>
            <th>Домен</th>
            <th>Subject</th>
            <th>Issuer</th>
            <th>Not Before</th>
            <th>Not After</th>
            <th>Статус</th>
        </tr>
        {% for cert in web_cert_info.results %}
        {% set not_after = cert.stdout | regex_search('notAfter=(.*)') %}
        {% set days_left = (not_after[1] | to_datetime('%b %d %H:%M:%S %Y %Z') - ansible_date_time.iso8601 | to_datetime('%Y-%m-%dT%H:%M:%SZ')).days %}
        <tr class="{% if days_left < critical_days %}critical{% elif days_left < warning_days %}warning{% else %}ok{% endif %}">
            <td>{{ cert.item.item }}</td>
            <td>{{ cert.stdout | regex_search('subject=(.*)') | default(['N/A'])[0] }}</td>
            <td>{{ cert.stdout | regex_search('issuer=(.*)') | default(['N/A'])[0] }}</td>
            <td>{{ cert.stdout | regex_search('notBefore=(.*)') | default(['N/A'])[0] }}</td>
            <td>{{ not_after | default(['N/A'])[0] }}</td>
            <td>
                {% if days_left < critical_days %}
                CRITICAL ({{ days_left }} дней осталось)
                {% elif days_left < warning_days %}
                WARNING ({{ days_left }} дней осталось)
                {% else %}
                OK ({{ days_left }} дней осталось)
                {% endif %}
            </td>
        </tr>
        {% endfor %}
    </table>

    <h2>Файловые сертификаты</h2>
    <table>
        <tr>
            <th>Путь</th>
            <th>Subject</th>
            <th>Not After</th>
            <th>Статус</th>
        </tr>
        {% for cert in file_cert_info.results %}
        {% set not_after = cert.stdout | regex_search('notAfter=(.*)') %}
        {% set days_left = (not_after[1] | to_datetime('%b %d %H:%M:%S %Y %Z') - ansible_date_time.iso8601 | to_datetime('%Y-%m-%dT%H:%M:%SZ')).days %}
        <tr class="{% if days_left < critical_days %}critical{% elif days_left < warning_days %}warning{% else %}ok{% endif %}">
            <td>{{ cert.item.path }}</td>
            <td>{{ cert.stdout | regex_search('subject=(.*)') | default(['N/A'])[0] }}</td>
            <td>{{ not_after | default(['N/A'])[0] }}</td>
            <td>
                {% if days_left < critical_days %}
                CRITICAL ({{ days_left }} дней осталось)
                {% elif days_left < warning_days %}
                WARNING ({{ days_left }} дней осталось)
                {% else %}
                OK ({{ days_left }} дней осталось)
                {% endif %}
            </td>
        </tr>
        {% endfor %}
    </table>

    <h2>Сертификаты служб</h2>
    <table>
        <tr>
            <th>Служба</th>
            <th>Хост:Порт</th>
            <th>Not After</th>
            <th>Статус</th>
        </tr>
        {% for cert in service_cert_info.results %}
        {% set not_after = cert.stdout | regex_search('notAfter=(.*)') %}
        {% set days_left = (not_after[1] | to_datetime('%b %d %H:%M:%S %Y %Z') - ansible_date_time.iso8601 | to_datetime('%Y-%m-%dT%H:%M:%SZ')).days %}
        <tr class="{% if days_left < critical_days %}critical{% elif days_left < warning_days %}warning{% else %}ok{% endif %}">
            <td>{{ cert.item.name }}</td>
            <td>{{ cert.item.host }}:{{ cert.item.port }}</td>
            <td>{{ not_after | default(['N/A'])[0] }}</td>
            <td>
                {% if days_left < critical_days %}
                CRITICAL ({{ days_left }} дней осталось)
                {% elif days_left < warning_days %}
                WARNING ({{ days_left }} дней осталось)
                {% else %}
                OK ({{ days_left }} дней осталось)
                {% endif %}
            </td>
        </tr>
        {% endfor %}
    </table>
</body>
</html>
```

## Использование роли

1. Создайте playbook:

```yaml
- hosts: all
  become: yes
  roles:
    - certificate_checker
  vars:
    cert_check_domains:
      - "example.com"
      - "api.example.com"
      - "{{ ansible_hostname }}"
    critical_days: 7
    warning_days: 30
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini check_certificates.yml
```

## Дополнительные возможности

1. **Уведомления**: Добавьте отправку email или сообщений в Slack при обнаружении критически близких к истечению сертификатов.
2. **Интеграция с мониторингом**: Экспорт данных в системы мониторинга (Prometheus, Zabbix).
3. **Автоматическое продление**: Интеграция с certbot для автоматического продления сертификатов Let's Encrypt.
4. **Проверка цепочки сертификатов**: Добавьте проверку всей цепочки сертификатов.
5. **Проверка отозванных сертификатов**: Интеграция с OCSP или CRL.

Эта роль предоставляет полный обзор состояния сертификатов на сервере с наглядным HTML-отчетом, классифицируя сертификаты по степени срочности их обновления.