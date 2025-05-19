Вот улучшенная Ansible роль для извлечения из PFX-файла:
- приватного ключа (private.key)
- сертификата (certificate.crt)
- промежуточных сертификатов (intermediate.pem)
- полного цепочного бандла (bundle.pem)

## Обновленная структура роли
```
roles/extract_pfx/
├── tasks/
│   └── main.yml
├── defaults/
│   └── main.yml
└── templates/
```

## Обновленный код роли

### defaults/main.yml
```yaml
---
# Исходный PFX файл
pfx_file: "certificate.pfx"
# Пароль от PFX файла
pfx_password: "changeit"
# Директория для сохранения извлеченных файлов
output_dir: "/etc/ssl/certs"
# Имена выходных файлов
private_key_name: "private.key"
certificate_name: "certificate.crt"
intermediate_name: "intermediate.pem"
bundle_name: "bundle.pem"
```

### tasks/main.yml
```yaml
---
- name: Ensure output directory exists
  file:
    path: "{{ output_dir }}"
    state: directory
    mode: 0755

- name: Extract private key from PFX (PEM format)
  command: >
    openssl pkcs12 -in "{{ pfx_file }}" -nocerts -nodes
    -out "{{ output_dir }}/{{ private_key_name }}"
    -password pass:{{ pfx_password }}
  args:
    creates: "{{ output_dir }}/{{ private_key_name }}"

- name: Extract certificate from PFX (PEM format)
  command: >
    openssl pkcs12 -in "{{ pfx_file }}" -clcerts -nokeys
    -out "{{ output_dir }}/{{ certificate_name }}"
    -password pass:{{ pfx_password }}
  args:
    creates: "{{ output_dir }}/{{ certificate_name }}"

- name: Extract intermediate certificates from PFX
  command: >
    openssl pkcs12 -in "{{ pfx_file }}" -cacerts -nokeys -chain
    -out "{{ output_dir }}/{{ intermediate_name }}"
    -password pass:{{ pfx_password }}
  args:
    creates: "{{ output_dir }}/{{ intermediate_name }}"

- name: Create certificate bundle (cert + intermediates)
  command: >
    cat "{{ output_dir }}/{{ certificate_name }}" "{{ output_dir }}/{{ intermediate_name }}"
    > "{{ output_dir }}/{{ bundle_name }}"
  args:
    creates: "{{ output_dir }}/{{ bundle_name }}"

- name: Set proper permissions for private key
  file:
    path: "{{ output_dir }}/{{ private_key_name }}"
    mode: 0600
    owner: root
    group: root

- name: Set proper permissions for certificate files
  file:
    path: "{{ output_dir }}/{{ item }}"
    mode: 0644
    owner: root
    group: root
  loop:
    - "{{ certificate_name }}"
    - "{{ intermediate_name }}"
    - "{{ bundle_name }}"
```

## Использование роли

1. Создайте playbook (например, `extract_pfx.yml`):
```yaml
---
- hosts: your_server
  become: yes
  roles:
    - extract_pfx
  vars:
    pfx_file: "/path/to/your/cert.pfx"
    pfx_password: "your_secure_password"
    output_dir: "/etc/ssl/your_domain"
```

2. Запустите playbook:
```bash
ansible-playbook extract_pfx.yml
```

## Особенности реализации:
1. Все файлы извлекаются в формате PEM
2. Bundle создается как объединение основного сертификата и промежуточных
3. Устанавливаются правильные права доступа:
   - 0600 для приватного ключа
   - 0644 для сертификатов
4. Добавлена идемпотентность - задачи выполняются только если файлы не существуют

Для работы роли требуется:
- OpenSSL на целевых хостах
- Правильные права на чтение PFX файла
- Достаточно места в целевой директории