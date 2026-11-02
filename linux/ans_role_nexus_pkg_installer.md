Расширим роль: она будет определять ОС и её версию через ansible_facts, затем сама выбирать нужный набор пакетов из конфигурации и использовать apt или dnf. Конфиг задается единым словарем packages_to_install с ключами по os_family и версии ОС.

📁 Новая структура роли

```text
roles/
└── nexus_pkg_installer/
    ├── tasks/
    │   ├── main.yml       # определение ОС и выбор пакетов
    │   ├── deb.yml        # для Debian/Ubuntu
    │   └── rpm.yml        # для RedHat/Oracle Linux
    └── defaults/
        └── main.yml
```

⚙️ defaults/main.yml

```yaml
# Базовый URL Nexus-репозитория
nexus_base_url: "http://nexus.company.local:8081/repository/packages-repo"

# Учетные данные (если требуется авторизация)
nexus_username: ""
nexus_password: ""

# Директория для временного хранения пакетов на сервере
pkg_download_dir: "/tmp/packages"

# Флаг: не проверять GPG для rpm (для неподписанных локальных пакетов)
rpm_disable_gpg_check: false

# Пустой словарь. Реальные значения задаются в inventory.
packages_to_install: {}
```

🧠 tasks/main.yml — диспетчер

```yaml
---
- name: Проверить, что ОС поддерживается
  ansible.builtin.assert:
    that:
      - ansible_facts['os_family'] in ['Debian', 'RedHat']
    fail_msg: >-
      ОС '{{ ansible_facts['distribution'] }}' не поддерживается.
      Поддерживаются: Debian/Ubuntu (family Debian) и
      RHEL/Oracle Linux (family RedHat).

- name: Получить набор пакетов для текущего семейства ОС
  ansible.builtin.set_fact:
    _os_family_pkgs: "{{ packages_to_install[ansible_facts['os_family']] | default({}) }}"

- name: Выбрать пакеты по версии (точная → major → пусто)
  ansible.builtin.set_fact:
    current_packages: >-
      {{
        _os_family_pkgs[ansible_facts['distribution_version']]
        | default(_os_family_pkgs[ansible_facts['distribution_major_version']], true)
        | default([], true)
      }}

- name: Отладка — какие пакеты будут установлены
  ansible.builtin.debug:
    msg: >-
      {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}
      → {{ current_packages | length }} пакет(ов): {{ current_packages | map(attribute='name') | list }}

- name: Установка deb-пакетов (Debian/Ubuntu)
  ansible.builtin.include_tasks: deb.yml
  when:
    - ansible_facts['os_family'] == 'Debian'
    - current_packages | length > 0

- name: Установка rpm-пакетов (RedHat/Oracle Linux)
  ansible.builtin.include_tasks: rpm.yml
  when:
    - ansible_facts['os_family'] == 'RedHat'
    - current_packages | length > 0
```

📦 tasks/deb.yml — Ubuntu 22.04 / 24.04

```yaml
---
- name: Создать директорию для deb-пакетов
  ansible.builtin.file:
    path: "{{ pkg_download_dir }}"
    state: directory
    mode: '0755'

- name: Скачать deb-пакеты с Nexus
  ansible.builtin.get_url:
    url: "{{ nexus_base_url }}/{{ item.path }}"
    dest: "{{ pkg_download_dir }}/{{ item.name }}"
    url_username: "{{ nexus_username | default(omit) }}"
    url_password: "{{ nexus_password | default(omit) }}"
    force_basic_auth: "{{ nexus_username | length > 0 }}"
    mode: '0644'
  loop: "{{ current_packages }}"
  loop_control:
    label: "{{ item.name }}"

- name: Установить deb-пакеты
  ansible.builtin.apt:
    deb: "{{ pkg_download_dir }}/{{ item.name }}"
    state: present
  loop: "{{ current_packages }}"
  loop_control:
    label: "{{ item.name }}"
```

📦 tasks/rpm.yml — Oracle Linux 8 / 9

```yaml
---
- name: Создать директорию для rpm-пакетов
  ansible.builtin.file:
    path: "{{ pkg_download_dir }}"
    state: directory
    mode: '0755'

- name: Скачать rpm-пакеты с Nexus
  ansible.builtin.get_url:
    url: "{{ nexus_base_url }}/{{ item.path }}"
    dest: "{{ pkg_download_dir }}/{{ item.name }}"
    url_username: "{{ nexus_username | default(omit) }}"
    url_password: "{{ nexus_password | default(omit) }}"
    force_basic_auth: "{{ nexus_username | length > 0 }}"
    mode: '0644'
  loop: "{{ current_packages }}"
  loop_control:
    label: "{{ item.name }}"

- name: Установить rpm-пакеты (Oracle Linux)
  ansible.builtin.dnf:
    name: "{{ pkg_download_dir }}/{{ item.name }}"
    state: present
    disable_gpg_check: "{{ rpm_disable_gpg_check }}"
  loop: "{{ current_packages }}"
  loop_control:
    label: "{{ item.name }}"
```

🖥️ Инвентарь: свой набор пакетов для каждой ОС/версии

Ключевая идея — один словарь packages_to_install, внутри которого пакеты разложены по os_family и версии. Роль сама выберет нужное (сначала пробует точную версию, затем major).

inventory/hosts.yml:

```yaml
all:
  vars:
    nexus_base_url: "http://nexus.company.local:8081/repository/packages-repo"
    nexus_username: "deploy_user"
    nexus_password: "secret"
    rpm_disable_gpg_check: true

    packages_to_install:
      # === Ubuntu (os_family = Debian) ===
      Debian:
        "22.04":
          - name: "app_1.0.0_amd64.deb"
            path: "pool/main/a/app/app_1.0.0_amd64.deb"
          - name: "agent_1.0.0_amd64.deb"
            path: "pool/main/a/agent/agent_1.0.0_amd64.deb"
        "24.04":
          - name: "app_2.0.0_amd64.deb"
            path: "pool/main/a/app/app_2.0.0_amd64.deb"
          - name: "agent_2.0.0_amd64.deb"
            path: "pool/main/a/agent/agent_2.0.0_amd64.deb"

      # === Oracle Linux (os_family = RedHat) ===
      RedHat:
        "8":
          - name: "app_1.0.0-1.el8.x86_64.rpm"
            path: "pool/main/a/app/app_1.0.0-1.el8.x86_64.rpm"
          - name: "agent_1.0.0-1.el8.x86_64.rpm"
            path: "pool/main/a/agent/agent_1.0.0-1.el8.x86_64.rpm"
        "9":
          - name: "app_2.0.0-1.el9.x86_64.rpm"
            path: "pool/main/a/app/app_2.0.0-1.el9.x86_64.rpm"
          - name: "agent_2.0.0-1.el9.x86_64.rpm"
            path: "pool/main/a/agent/agent_2.0.0-1.el9.x86_64.rpm"

  children:
    web_servers:
      hosts:
        web-ubuntu-2204:
          ansible_host: 192.168.1.10

        web-ubuntu-2404:
          ansible_host: 192.168.1.11

        web-ol-8:
          ansible_host: 192.168.1.20

        web-ol-9:
          ansible_host: 192.168.1.21

    # Пример переопределения на уровне конкретного хоста
    db_servers:
      hosts:
        db-ubuntu-2404:
          ansible_host: 192.168.1.30
          packages_to_install:
            Debian:
              "24.04":
                - name: "postgresql_16_amd64.deb"
                  path: "pool/main/p/postgresql/postgresql_16_amd64.deb"
```

🚀 Плейбук

```yaml
---
- name: Установка пакетов из Nexus
  hosts: all
  become: true
  roles:
    - nexus_pkg_installer
```

💡 Как это работает и важные нюансы

Выбор пакетов по ОС и версии:

· ansible_facts['os_family'] для Ubuntu → Debian, для Oracle Linux → RedHat.
· Сначала ищется точное совпадение по ansible_facts['distribution_version'] (например, "22.04"), затем по distribution_major_version ("22", "24", "8", "9").
· Поэтому в конфиге можно писать либо "22.04", либо "22" — обе формы будут корректно найдены.

Установка:

· Для Debian/Ubuntu — ansible.builtin.apt с параметром deb.
· Для Oracle Linux 8/9 — ansible.builtin.dnf. Модуль идемпотентен и не переустанавливает уже установленное.
· rpm_disable_gpg_check: true нужен для неподписанных локальных rpm-пакетов из Nexus. Если пакеты подписаны — оставьте false.

Аутентификация:

· При пустом nexus_username параметры url_username/url_password будут опущены (omit), а force_basic_auth выключен.
· Если в Nexus включена авторизация — заполните nexus_username/nexus_password в inventory.

Гибкость конфигурации:

· В group_vars/all задаются пакеты по умолчанию для всех ОС.
· Для конкретной группы или хоста можно переопределить packages_to_install — как показано для db-ubuntu-2404.
· Если нужной версии ОС нет в конфиге — роль просто пропустит установку с отладочным сообщением (см. задачу debug), ошибки не будет.

Дальнейшее развитие:

· Если понадобится различать RHEL и OracleLinux (сейчас оба — RedHat), замените в роли os_family на ansible_facts['distribution'] и используйте ключи Ubuntu, OracleLinux.