# Ansible роль для извлечения информации из PFX сертификата

Роль будет извлекать следующую информацию из PFX файла:
- Common Name (CN)
- Distinguished Name (DN)
- DNS имена (Subject Alternative Names)
- Даты начала и окончания действия сертификата

## Структура роли

```
roles/pfx_info/
├── tasks/
│   └── main.yml
├── defaults/
│   └── main.yml
└── templates/
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Исходный PFX файл
pfx_file: "/path/to/certificate.pfx"
# Пароль для PFX файла
pfx_password: "your_password"
# Временный файл для извлеченного сертификата
temp_cert_file: "/tmp/cert_temp.crt"
```

### tasks/main.yml

```yaml
---
- name: Проверить существование PFX файла
  ansible.builtin.stat:
    path: "{{ pfx_file }}"
  register: pfx_stat

- name: Извлечь сертификат из PFX во временный файл
  ansible.builtin.command: >
    openssl pkcs12 -in "{{ pfx_file }}" -clcerts -nokeys -out "{{ temp_cert_file }}"
    -password pass:{{ pfx_password }} -nodes
  when: pfx_stat.stat.exists
  register: extract_cert
  changed_when: extract_cert.rc == 0

- name: Получить информацию о сертификате
  ansible.builtin.command: >
    openssl x509 -in "{{ temp_cert_file }}" -noout -text
  when: extract_cert.rc == 0
  register: cert_info
  changed_when: false

- name: Парсинг информации о сертификате
  ansible.builtin.set_fact:
    cert_details: >-
      {{
        {
          'subject': cert_info.stdout | regex_findall('Subject: (.*?)(?:\n|$)'),
          'issuer': cert_info.stdout | regex_findall('Issuer: (.*?)(?:\n|$)'),
          'not_before': cert_info.stdout | regex_findall('Not Before: (.*?)(?:\n|$)'),
          'not_after': cert_info.stdout | regex_findall('Not After : (.*?)(?:\n|$)'),
          'dns_names': cert_info.stdout | regex_findall('DNS:([^,\n]*)'),
          'cn': (cert_info.stdout | regex_findall('Subject:.*CN=([^,/]*)'))[0] | default('')
        }
      }}

- name: Вывести информацию о сертификате
  ansible.builtin.debug:
    msg: |
      PFX Certificate Information:
      - Common Name (CN): {{ cert_details.cn }}
      - Distinguished Name (DN): {{ cert_details.subject[0] }}
      - Issuer: {{ cert_details.issuer[0] }}
      - Valid from: {{ cert_details.not_before[0] }}
      - Valid to: {{ cert_details.not_after[0] }}
      - DNS Names: {{ cert_details.dns_names | join(', ') }}

- name: Удалить временный файл сертификата
  ansible.builtin.file:
    path: "{{ temp_cert_file }}"
    state: absent
```

## Использование роли

1. Создайте playbook (например, `get_pfx_info.yml`):

```yaml
---
- hosts: localhost
  roles:
    - pfx_info
  vars:
    pfx_file: "/path/to/your/cert.pfx"
    pfx_password: "your_password"
```

2. Запустите playbook:

```bash
ansible-playbook get_pfx_info.yml
```

## Альтернативный вариант с использованием модуля community.crypto

Если у вас установлен модуль `community.crypto`, можно использовать более надежный способ:

```yaml
- name: Get certificate info using community.crypto
  community.crypto.x509_certificate_info:
    path: "{{ temp_cert_file }}"
  register: cert_info

- name: Display parsed info
  ansible.builtin.debug:
    msg: |
      Certificate Details:
      - Subject: {{ cert_info.subject }}
      - Issuer: {{ cert_info.issuer }}
      - CN: {{ cert_info.subject.common_name }}
      - SANs: {{ cert_info.extensions.subject_alt_name.dns_names | default([]) | join(', ') }}
      - Valid from: {{ cert_info.not_valid_before }}
      - Valid to: {{ cert_info.not_valid_after }}
```

Для использования этого варианта сначала установите модуль:
```bash
ansible-galaxy collection install community.crypto
```

Эта роль предоставит вам всю основную информацию о сертификате, содержащемся в PFX файле.