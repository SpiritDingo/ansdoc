# Роль Ansible для установки и настройки HashiCorp Vault

Вот пример роли Ansible для установки и настройки HashiCorp Vault. Роль включает установку Vault, базовую конфигурацию, инициализацию и настройку systemd для управления сервисом.

## Структура роли

```
roles/vault/
├── defaults
│   └── main.yml
├── files
│   ├── vault.hcl.j2
│   └── vault.service.j2
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    └── vault.json.j2
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Версия Vault для установки
vault_version: "1.15.0"
vault_architecture: "amd64"

# Пользователь и группа для запуска Vault
vault_user: vault
vault_group: vault

# Директории Vault
vault_config_dir: "/etc/vault.d"
vault_data_dir: "/var/lib/vault"
vault_bin_dir: "/usr/local/bin"
vault_log_dir: "/var/log/vault"

# Настройки конфигурации Vault
vault_config:
  listener:
    tcp:
      address: "0.0.0.0:8200"
      tls_disable: true
  storage:
    file:
      path: "{{ vault_data_dir }}"
  ui: true
  api_addr: "http://0.0.0.0:8200"
  cluster_addr: "https://0.0.0.0:8201"

# Настройки инициализации
vault_initialize: true
vault_seal_keys: []
vault_root_token: ""
```

### tasks/main.yml

```yaml
---
- name: Установка зависимостей
  apt:
    name:
      - curl
      - gnupg
      - software-properties-common
    state: present
  when: ansible_os_family == 'Debian'

- name: Добавление GPG ключа HashiCorp
  apt_key:
    url: "https://apt.releases.hashicorp.com/gpg"
    state: present
  when: ansible_os_family == 'Debian'

- name: Добавление репозитория HashiCorp
  apt_repository:
    repo: "deb [arch=amd64] https://apt.releases.hashicorp.com {{ ansible_distribution_release }} main"
    state: present
    update_cache: yes
  when: ansible_os_family == 'Debian'

- name: Установка Vault
  apt:
    name: vault
    state: present
  when: ansible_os_family == 'Debian'

- name: Создание пользователя и группы Vault
  user:
    name: "{{ vault_user }}"
    group: "{{ vault_group }}"
    system: yes
    create_home: no
    shell: /bin/false

- name: Создание директорий для Vault
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ vault_user }}"
    group: "{{ vault_group }}"
    mode: 0750
  loop:
    - "{{ vault_config_dir }}"
    - "{{ vault_data_dir }}"
    - "{{ vault_log_dir }}"

- name: Копирование конфигурационного файла Vault
  template:
    src: vault.hcl.j2
    dest: "{{ vault_config_dir }}/vault.hcl"
    owner: "{{ vault_user }}"
    group: "{{ vault_group }}"
    mode: 0640
  notify: restart vault

- name: Копирование unit-файла systemd для Vault
  template:
    src: vault.service.j2
    dest: /etc/systemd/system/vault.service
    owner: root
    group: root
    mode: 0644
  notify:
    - daemon-reload
    - restart vault

- name: Включение и запуск сервиса Vault
  systemd:
    name: vault
    state: started
    enabled: yes
    daemon_reload: yes

- name: Инициализация Vault (если требуется)
  command: >
    vault operator init -key-shares=1 -key-threshold=1 -format=json
  register: vault_init
  environment:
    VAULT_ADDR: "http://127.0.0.1:8200"
  changed_when: false
  when: vault_initialize and vault_root_token == ""

- name: Сохранение ключей и root token
  set_fact:
    vault_seal_keys: "{{ (vault_init.stdout | from_json).keys }}"
    vault_root_token: "{{ (vault_init.stdout | from_json).root_token }}"
  when: vault_init is defined and vault_init.stdout != ""

- name: Разблокировка Vault (если есть ключи)
  command: >
    vault operator unseal {{ item }}
  environment:
    VAULT_ADDR: "http://127.0.0.1:8200"
  with_items: "{{ vault_seal_keys }}"
  when: vault_seal_keys | length > 0

- name: Вход с root token (если есть токен)
  command: >
    vault login {{ vault_root_token }}
  environment:
    VAULT_ADDR: "http://127.0.0.1:8200"
  when: vault_root_token != ""
```

### handlers/main.yml

```yaml
---
- name: restart vault
  systemd:
    name: vault
    state: restarted
    daemon_reload: yes

- name: daemon-reload
  systemd:
    daemon_reload: yes
```

### templates/vault.hcl.j2

```hcl
storage "file" {
  path = "{{ vault_data_dir }}"
}

listener "tcp" {
  address     = "{{ vault_config.listener.tcp.address }}"
  tls_disable = {{ vault_config.listener.tcp.tls_disable | lower }}
}

ui = {{ vault_config.ui | lower }}

api_addr = "{{ vault_config.api_addr }}"
cluster_addr = "{{ vault_config.cluster_addr }}"
```

### templates/vault.service.j2

```ini
[Unit]
Description="HashiCorp Vault - A tool for managing secrets"
Documentation=https://www.vaultproject.io/docs/
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty={{ vault_config_dir }}/vault.hcl

[Service]
User={{ vault_user }}
Group={{ vault_group }}
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
Capabilities=CAP_IPC_LOCK+ep
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart={{ vault_bin_dir }}/vault server -config={{ vault_config_dir }}/vault.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
StartLimitInterval=60
StartLimitBurst=3
LimitNOFILE=65536
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target
```

## Использование роли

1. Создайте playbook (например, `vault.yml`):

```yaml
---
- hosts: vault_servers
  become: true
  roles:
    - vault
```

2. Создайте inventory файл или используйте существующий, указав группу `vault_servers`.

3. Запустите playbook:

```bash
ansible-playbook -i inventory vault.yml
```

## Дополнительные возможности

1. **Конфигурация TLS**: Добавьте параметры TLS в конфигурацию Vault.
2. **Интеграция с Consul**: Для production используйте Consul в качестве storage backend.
3. **Auto-unseal**: Настройте автоматическое распечатывание с помощью KMS.
4. **HA кластер**: Настройте несколько серверов Vault в режиме высокой доступности.

Эта роль предоставляет базовую установку Vault. Для production окружений рекомендуется дополнительно настроить TLS, бэкеп хранилища и политики доступа.