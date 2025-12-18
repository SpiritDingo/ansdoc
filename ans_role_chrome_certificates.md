Вот пример Ansible роли для добавления локальных корневых и промежуточных сертификатов в Google Chrome на Linux системах:

Структура роли:

```
roles/chrome-certificates/
├── tasks/
│   └── main.yml
├── files/
│   ├── root_ca.crt      # Ваш корневой сертификат
│   └── intermediate.crt  # Ваш промежуточный сертификат
├── defaults/
│   └── main.yml
└── meta/
    └── main.yml
```

1. defaults/main.yml

```yaml
---
# Имена файлов сертификатов
chrome_certificates:
  root_ca: "root_ca.crt"
  intermediate: "intermediate.crt"

# Путь для копирования сертификатов
certificates_dest: "/usr/local/share/ca-certificates"

# Имена для сертификатов в системе
certificate_names:
  root: "my-root-ca.crt"
  intermediate: "my-intermediate-ca.crt"

# Обновлять хранилище сертификатов
update_ca_certificates: true

# Перезапускать Chrome после добавления сертификатов
restart_chrome: false
```

2. tasks/main.yml

```yaml
---
- name: Install required packages
  apt:
    name:
      - ca-certificates
      - libnss3-tools
    state: present
    update_cache: yes
  when: ansible_os_family == "Debian"

- name: Install required packages for RHEL
  yum:
    name:
      - ca-certificates
      - nss-tools
    state: present
  when: ansible_os_family == "RedHat"

- name: Create certificates directory
  file:
    path: "{{ certificates_dest }}"
    state: directory
    mode: '0755'

- name: Copy root CA certificate
  copy:
    src: "{{ chrome_certificates.root_ca }}"
    dest: "{{ certificates_dest }}/{{ certificate_names.root }}"
    mode: '0644'
    owner: root
    group: root

- name: Copy intermediate certificate
  copy:
    src: "{{ chrome_certificates.intermediate }}"
    dest: "{{ certificates_dest }}/{{ certificate_names.intermediate }}"
    mode: '0644'
    owner: root
    group: root

- name: Update CA certificates store
  command: update-ca-certificates
  when: update_ca_certificates

- name: Add certificates to NSS database (system-wide)
  command: >
    certutil -A -n "{{ item.name }}" -t "CT,C,C" -i "{{ certificates_dest }}/{{ item.file }}"
    -d sql:/etc/pki/nssdb
  with_items:
    - { name: "My Root CA", file: "{{ certificate_names.root }}" }
    - { name: "My Intermediate CA", file: "{{ certificate_names.intermediate }}" }
  when: ansible_os_family == "RedHat"

- name: Add certificates to Firefox NSS database
  command: >
    certutil -A -n "{{ item.name }}" -t "CT,C,C" -i "{{ certificates_dest }}/{{ item.file }}"
    -d /usr/share/firefox
  with_items:
    - { name: "My Root CA", file: "{{ certificate_names.root }}" }
    - { name: "My Intermediate CA", file: "{{ certificate_names.intermediate }}" }
  when: ansible_os_family == "Debian"

- name: Find Chrome NSS databases for all users
  find:
    paths: "/home"
    patterns: "*.db"
    file_type: file
    recurse: yes
    hidden: yes
  register: chrome_dbs

- name: Add certificates to Chrome NSS databases
  command: >
    certutil -A -n "{{ item.name }}" -t "CT,C,C" -i "{{ certificates_dest }}/{{ item.cert_file }}"
    -d "sql:{{ item.db_path | dirname }}"
  with_items: "{{ chrome_dbs.files }}"
  loop_control:
    loop_var: db_file
  vars:
    db_path: "{{ db_file.path }}"
    certs_to_add:
      - { name: "My Root CA", cert_file: "{{ certificate_names.root }}" }
      - { name: "My Intermediate CA", cert_file: "{{ certificate_names.intermediate }}" }
  when: 
    - "'chrome' in db_path or 'google-chrome' in db_path"
    - "'CertDB' in db_path"

- name: Create Chrome policies for certificate trust
  template:
    src: chrome_policies.json.j2
    dest: /etc/opt/chrome/policies/managed/chrome_certificates.json
    mode: '0644'
  when: ansible_os_family == "Debian"

- name: Create Chrome policies for certificate trust (RHEL)
  template:
    src: chrome_policies.json.j2
    dest: /etc/chromium/policies/managed/chrome_certificates.json
    mode: '0644'
  when: ansible_os_family == "RedHat"

- name: Restart Chrome if running
  shell: |
    pkill -f chrome
  ignore_errors: yes
  when: restart_chrome
```

3. templates/chrome_policies.json.j2

```json
{
  "AutoSelectCertificateForUrls": [
    {
      "pattern": "https://your-domain.com",
      "filter": {
        "ISSUER": {
          "CN": "Your Certificate Authority"
        }
      }
    }
  ],
  "SecurityTokenSessionNotificationSeconds": 0
}
```

4. Пример playbook (chrome_certificates.yml)

```yaml
---
- hosts: all
  become: yes
  roles:
    - chrome-certificates
  vars:
    chrome_certificates:
      root_ca: "{{ playbook_dir }}/files/root_ca.crt"
      intermediate: "{{ playbook_dir }}/files/intermediate.crt"
    certificate_names:
      root: "company-root-ca.crt"
      intermediate: "company-intermediate-ca.crt"
    restart_chrome: true
```

5. Альтернативный вариант - через пользовательские профили Chrome

Добавьте этот task в tasks/main.yml для работы с пользовательскими профилями:

```yaml
- name: Add certificates to user Chrome profiles
  block:
    - name: Get list of users with home directories
      shell: |
        getent passwd | awk -F: '$6 ~ /^\/home/ {print $1":"$6}'
      register: users_list

    - name: Add certificates to each user's Chrome NSS database
      command: >
        certutil -A -n "{{ item.1.name }}" -t "CT,C,C" -i "{{ certificates_dest }}/{{ item.1.cert_file }}"
        -d "sql:{{ item.0.split(':')[1] }}/.pki/nssdb"
      with_nested:
        - "{{ users_list.stdout_lines }}"
        - "{{ certs_to_add }}"
      vars:
        certs_to_add:
          - { name: "My Root CA", cert_file: "{{ certificate_names.root }}" }
          - { name: "My Intermediate CA", cert_file: "{{ certificate_names.intermediate }}" }
      loop_control:
        label: "{{ item.0.split(':')[0] }} - {{ item.1.name }}"
```

Использование:

1. Положите ваши сертификаты в files/ директорию роли
2. Настройте переменные в defaults/main.yml
3. Запустите playbook:

```bash
ansible-playbook -i inventory chrome_certificates.yml -K
```

Примечания:

1. Роль работает для Debian/Ubuntu и RHEL/CentOS систем
2. Chrome использует системное хранилище NSS для сертификатов
3. Для применения изменений может потребоваться перезапуск Chrome
4. Для корпоративного использования рекомендуется настроить Chrome через политики
5. Убедитесь, что сертификаты в PEM формате (Base64)

Эта роль добавляет сертификаты в системное хранилище и в NSS базы Chrome для всех пользователей, обеспечивая доверие к вашим локальным CA.