Вот пример Ansible роли для извлечения сертификата (.crt) и приватного ключа (.key) из PFX-файла:

Структура роли:

```bash
roles/extract_pfx/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
```

1. defaults/main.yml:

```yaml
pfx_path: "/path/to/cert.pfx"
pfx_password: "your_password"
output_directory: "/etc/ssl"
cert_file: "cert.crt"
key_file: "private.key"
```

2. tasks/main.yml:

```yaml
---
- name: Check if PFX file exists
  stat:
    path: "{{ pfx_path }}"
  register: pfx_stat

- name: Create output directory
  file:
    path: "{{ output_directory }}"
    state: directory
    mode: 0755
  when: pfx_stat.stat.exists

- name: Extract certificate from PFX
  community.crypto.openssl_pkcs12:
    path: "{{ pfx_path }}"
    password: "{{ pfx_password }}"
    cert_path: "{{ output_directory }}/{{ cert_file }}"
    state: present
  when: pfx_stat.stat.exists
  no_log: true  # Скрывает вывод с паролем

- name: Extract private key from PFX
  community.crypto.openssl_pkcs12:
    path: "{{ pfx_path }}"
    password: "{{ pfx_password }}"
    privatekey_path: "{{ output_directory }}/{{ key_file }}"
    state: present
  when: pfx_stat.stat.exists
  no_log: true
```

3. Использование в playbook:

```yaml
- hosts: all
  roles:
    - extract_pfx
  vars:
    pfx_path: "/tmp/mycert.pfx"
    pfx_password: "{{ vault_pfx_password }}"
    output_directory: "/etc/nginx/ssl"
```

Для безопасности:

1. Используйте Ansible Vault для шифрования пароля:

```bash
ansible-vault encrypt_string 'your_password' --name 'vault_pfx_password'
```

1. Добавьте в playbook:

```yaml
vars_files:
  - vault.yml
```

Требования:

· Установленная коллекция community.crypto:

```bash
ansible-galaxy collection install community.crypto
```

· На целевых хостах должен быть установлен OpenSSL

Проверка извлеченных файлов:

```yaml
- name: Validate certificate
  command: openssl x509 -noout -in "{{ output_directory }}/{{ cert_file }}"
  
- name: Validate private key
  command: openssl rsa -check -noout -in "{{ output_directory }}/{{ key_file }}"
```

Эта роль:

1. Проверяет существование PFX-файла
2. Создает выходную директорию
3. Извлекает сертификат и ключ в отдельные файлы
4. Использует безопасное хранение паролей через Ansible Vault
5. Проверяет валидность полученных файлов