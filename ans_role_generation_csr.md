# Генерация приватного ключа и CSR с помощью Ansible

Вот пример Ansible роли для генерации приватного ключа и CSR (Certificate Signing Request):

## Структура роли

```
roles/generate_csr/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    └── csr.conf.j2
```

## Файлы роли

### `defaults/main.yml`

```yaml
---
# Настройки по умолчанию
csr_output_dir: "/etc/ssl/certs"
key_size: 2048
key_file: "{{ csr_output_dir }}/{{ common_name }}.key"
csr_file: "{{ csr_output_dir }}/{{ common_name }}.csr"
days_valid: 365

# Параметры CSR
country_name: "US"
state_or_province_name: "California"
locality_name: "San Francisco"
organization_name: "Example Inc"
organizational_unit_name: "IT Department"
common_name: "example.com"
email_address: "admin@example.com"
subject_alt_names: []
```

### `templates/csr.conf.j2`

```jinja2
[ req ]
default_bits       = {{ key_size }}
prompt             = no
default_md         = sha256
distinguished_name = req_distinguished_name
req_extensions     = req_ext

[ req_distinguished_name ]
C            = {{ country_name }}
ST           = {{ state_or_province_name }}
L            = {{ locality_name }}
O            = {{ organization_name }}
OU           = {{ organizational_unit_name }}
CN           = {{ common_name }}
emailAddress = {{ email_address }}

[ req_ext ]
{% if subject_alt_names %}
subjectAltName = @alt_names

[ alt_names ]
{% for san in subject_alt_names %}
DNS.{{ loop.index }} = {{ san }}
{% endfor %}
{% endif %}
```

### `tasks/main.yml`

```yaml
---
- name: Ensure output directory exists
  ansible.builtin.file:
    path: "{{ csr_output_dir }}"
    state: directory
    mode: '0755'

- name: Generate CSR configuration file
  ansible.builtin.template:
    src: csr.conf.j2
    dest: "/tmp/{{ common_name }}.csr.conf"
    mode: '0644'

- name: Generate private key
  community.crypto.openssl_privatekey:
    path: "{{ key_file }}"
    size: "{{ key_size }}"
    type: RSA
    mode: '0600'

- name: Generate CSR
  community.crypto.openssl_csr:
    path: "{{ csr_file }}"
    privatekey_path: "{{ key_file }}"
    csr_config: "/tmp/{{ common_name }}.csr.conf"
    mode: '0644'

- name: Clean up temporary CSR config
  ansible.builtin.file:
    path: "/tmp/{{ common_name }}.csr.conf"
    state: absent
```

## Использование роли

Пример playbook для использования этой роли:

```yaml
---
- hosts: localhost
  vars:
    common_name: "mysite.example.com"
    subject_alt_names:
      - "www.mysite.example.com"
      - "api.mysite.example.com"
  roles:
    - generate_csr
```

## Требования

Для работы этой роли потребуются:
1. Установленный OpenSSL на целевом хосте
2. Коллекция `community.crypto` (устанавливается через `ansible-galaxy collection install community.crypto`)

## Дополнительные возможности

Вы можете расширить эту роль, добавив:
1. Генерацию самоподписанного сертификата
2. Загрузку CSR в центр сертификации
3. Проверку существующих ключей перед генерацией
4. Поддержку разных типов ключей (ECDSA и т.д.)