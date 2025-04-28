# Ansible роль для установки и настройки центра выпуска сертификатов (CA) на Linux

Эта роль устанавливает и настраивает приватный центр сертификации на базе OpenSSL.

## Структура роли

```
roles/ca_server/
├── defaults
│   └── main.yml
├── files
│   ├── openssl.cnf
│   └── index.txt
├── tasks
│   ├── main.yml
│   ├── install.yml
│   ├── configure.yml
│   └── post_install.yml
├── templates
│   └── openssl.cnf.j2
└── handlers
    └── main.yml
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Основные параметры CA
ca_name: "MyPrivateCA"
ca_country: "RU"
ca_state: "Moscow"
ca_locality: "Moscow"
ca_organization: "My Organization"
ca_organizational_unit: "IT Department"
ca_email: "ca@example.com"
ca_days: 3650  # Срок действия корневого сертификата

# Пути для CA
ca_root_dir: "/etc/ssl/ca"
ca_certs_dir: "{{ ca_root_dir }}/certs"
ca_crl_dir: "{{ ca_root_dir }}/crl"
ca_new_certs_dir: "{{ ca_root_dir }}/newcerts"
ca_private_dir: "{{ ca_root_dir }}/private"
ca_serial_file: "{{ ca_root_dir }}/serial"
ca_database_file: "{{ ca_root_dir }}/index.txt"
ca_crl_file: "{{ ca_root_dir }}/crl.pem"

# Параметры OpenSSL
openssl_config: "{{ ca_root_dir }}/openssl.cnf"
openssl_key_size: 4096
```

### tasks/main.yml

```yaml
---
- name: Include installation tasks
  include_tasks: install.yml

- name: Include configuration tasks
  include_tasks: configure.yml

- name: Include post-installation tasks
  include_tasks: post_install.yml
```

### tasks/install.yml

```yaml
---
- name: Install required packages
  package:
    name:
      - openssl
      - openssl-perl
    state: present

- name: Create CA directory structure
  file:
    path: "{{ item }}"
    state: directory
    owner: root
    group: root
    mode: '0750'
  loop:
    - "{{ ca_root_dir }}"
    - "{{ ca_certs_dir }}"
    - "{{ ca_crl_dir }}"
    - "{{ ca_new_certs_dir }}"
    - "{{ ca_private_dir }}"

- name: Initialize CA files
  file:
    path: "{{ item.path }}"
    state: touch
    owner: root
    group: root
    mode: '{{ item.mode }}'
  loop:
    - { path: "{{ ca_database_file }}", mode: '0640' }
    - { path: "{{ ca_serial_file }}", mode: '0640' }
    - { path: "{{ ca_root_dir }}/crlnumber", mode: '0640' }

- name: Set initial serial number
  copy:
    content: "1000\n"
    dest: "{{ ca_serial_file }}"
    owner: root
    group: root
    mode: '0640'
```

### tasks/configure.yml

```yaml
---
- name: Deploy OpenSSL configuration
  template:
    src: openssl.cnf.j2
    dest: "{{ openssl_config }}"
    owner: root
    group: root
    mode: '0640'

- name: Generate CA private key
  openssl_privatekey:
    path: "{{ ca_private_dir }}/ca.key"
    size: "{{ openssl_key_size }}"
    type: RSA
    mode: '0400'

- name: Generate CA root certificate
  openssl_certificate:
    path: "{{ ca_root_dir }}/ca.crt"
    privatekey_path: "{{ ca_private_dir }}/ca.key"
    csr_path: "{{ ca_root_dir }}/ca.csr"
    provider: selfsigned
    selfsigned_digest: sha256
    selfsigned_not_after: "+{{ ca_days }}d"
    selfsigned_not_before: "-1d"
    country_name: "{{ ca_country }}"
    state_or_province_name: "{{ ca_state }}"
    locality_name: "{{ ca_locality }}"
    organization_name: "{{ ca_organization }}"
    organizational_unit_name: "{{ ca_organizational_unit }}"
    common_name: "{{ ca_name }}"
    email_address: "{{ ca_email }}"
    basic_constraints_critical: yes
    basic_constraints: "CA:TRUE"
    key_usage_critical: yes
    key_usage: "keyCertSign, cRLSign"
    subject_key_identifier: "hash"
    authority_key_identifier: "keyid:always"
```

### tasks/post_install.yml

```yaml
---
- name: Create initial CRL
  command: "openssl ca -gencrl -config {{ openssl_config }} -out {{ ca_crl_file }}"
  become: yes

- name: Set permissions on CRL file
  file:
    path: "{{ ca_crl_file }}"
    mode: '0644'
    owner: root
    group: root

- name: Create symbolic link to system certs
  file:
    src: "{{ ca_root_dir }}/ca.crt"
    dest: "/usr/local/share/ca-certificates/{{ ca_name }}.crt"
    state: link
  when: ansible_os_family == 'Debian'

- name: Update CA certificates (Debian)
  command: update-ca-certificates
  when: ansible_os_family == 'Debian'

- name: Copy CA cert to system store (RedHat)
  copy:
    src: "{{ ca_root_dir }}/ca.crt"
    dest: "/etc/pki/ca-trust/source/anchors/{{ ca_name }}.crt"
  when: ansible_os_family == 'RedHat'

- name: Update CA trust (RedHat)
  command: update-ca-trust
  when: ansible_os_family == 'RedHat'
```

### templates/openssl.cnf.j2

```ini
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = {{ ca_root_dir }}
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE         = $dir/private/.rand
private_key       = $dir/private/ca.key
certificate       = $dir/ca.crt
crl               = $dir/crl.pem
crlnumber         = $dir/crlnumber
default_days      = 365
default_crl_days  = 30
default_md        = sha256
preserve          = no
policy            = policy_anything

[ policy_anything ]
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
default_bits        = {{ openssl_key_size }}
default_keyfile     = {{ ca_private_dir }}/ca.key
distinguished_name  = req_distinguished_name
x509_extensions     = v3_ca
string_mask         = utf8only
prompt              = no

[ req_distinguished_name ]
countryName             = {{ ca_country }}
stateOrProvinceName     = {{ ca_state }}
localityName            = {{ ca_locality }}
organizationName        = {{ ca_organization }}
organizationalUnitName  = {{ ca_organizational_unit }}
commonName              = {{ ca_name }}
emailAddress            = {{ ca_email }}

[ v3_ca ]
subjectKeyIdentifier    = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints       = critical, CA:true
keyUsage               = critical, digitalSignature, keyCertSign, cRLSign

[ server_cert ]
basicConstraints       = CA:FALSE
nsCertType             = server
nsComment              = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage               = critical, digitalSignature, keyEncipherment
extendedKeyUsage       = serverAuth

[ client_cert ]
basicConstraints       = CA:FALSE
nsCertType             = client
nsComment              = "OpenSSL Generated Client Certificate"
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage               = critical, digitalSignature
extendedKeyUsage       = clientAuth
```

## Пример использования

1. Создайте playbook `setup_ca.yml`:

```yaml
---
- hosts: ca_server
  become: yes
  roles:
    - ca_server
  vars:
    ca_name: "CompanyInternalCA"
    ca_country: "RU"
    ca_state: "Moscow"
    ca_locality: "Moscow"
    ca_organization: "My Company"
    ca_organizational_unit: "IT Security"
    ca_email: "ca-admin@company.com"
    ca_days: 3650
```

2. Запустите playbook:
```bash
ansible-playbook -i inventory setup_ca.yml
```

## Дополнительные возможности

Для выпуска сертификатов можно добавить дополнительные задачи:

```yaml
- name: Generate CSR for server certificate
  openssl_csr:
    path: "/etc/ssl/certs/server.csr"
    privatekey_path: "/etc/ssl/private/server.key"
    country_name: "RU"
    organization_name: "My Company"
    common_name: "server.example.com"
    key_usage:
      - digitalSignature
      - keyEncipherment
    extended_key_usage:
      - serverAuth

- name: Sign server certificate
  openssl_certificate:
    path: "/etc/ssl/certs/server.crt"
    csr_path: "/etc/ssl/certs/server.csr"
    ownca_path: "{{ ca_root_dir }}/ca.crt"
    ownca_privatekey_path: "{{ ca_private_dir }}/ca.key"
    provider: ownca
    digest: sha256
    days: 365
    version: 3
```

Эта роль создает полнофункциональный центр сертификации, который можно использовать для:
- Выпуска серверных SSL/TLS сертификатов
- Выпуска клиентских сертификатов для аутентификации
- Создания CRL (Certificate Revocation List)
- Интеграции с системными хранилищами сертификатов