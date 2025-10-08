Вот универсальная **Ansible роль** для установки корневого сертификата в **Google Chrome** и **Firefox** на **Ubuntu** (и других системах на базе Debian/Ubuntu). Роль:

- Устанавливает `libnss3-tools`, если не установлен.
- Импортирует указанный сертификат в профили Chrome и всех пользователей Firefox.
- Поддерживает работу с несколькими пользователями.
- Позволяет настраивать имя сертификата, путь к файлу и настройки доверия.

---

### 📁 Структура роли

```
roles/
└── install_ca_cert/
    ├── defaults/main.yml
    ├── tasks/main.yml
    ├── vars/main.yml
    └── templates/
```

---

### 📄 `defaults/main.yml`

```yaml
---
# Путь к локальному файлу сертификата (на хосте Ansible)
ca_cert_src: "{{ role_path }}/files/ca.crt"

# Имя сертификата в хранилище
ca_cert_name: "Custom Root CA"

# Настройки доверия (по умолчанию — для SSL)
ca_cert_trust_flags: "TC,,"

# Пользователи, для которых нужно установить сертификат
ca_cert_users:
  - "{{ ansible_user | default('ubuntu') }}"

# Устанавливать в Chrome?
install_chrome: true

# Устанавливать в Firefox?
install_firefox: true
```

---

### 📄 `tasks/main.yml`

```yaml
---
- name: Установка libnss3-tools
  apt:
    name: libnss3-tools
    state: present
    update_cache: yes
  become: yes

- name: Убедиться, что файл сертификата существует на целевом хосте
  copy:
    src: "{{ ca_cert_src }}"
    dest: /tmp/{{ ca_cert_name | replace(' ', '_') }}.crt
    mode: '0644'
  register: cert_file

- name: Установка сертификата в Chrome для каждого пользователя
  command: >
    certutil -d sql:{{ item.home }}/.pki/nssdb
            -A -t "{{ ca_cert_trust_flags }}"
            -n "{{ ca_cert_name }}"
            -i /tmp/{{ ca_cert_name | replace(' ', '_') }}.crt
  args:
    creates: "{{ item.home }}/.pki/nssdb/cert9.db"
  loop: "{{ ca_cert_users_info }}"
  when: install_chrome | bool
  become: yes
  become_user: "{{ item.name }}"

- name: Установка сертификата в Firefox для каждого профиля каждого пользователя
  command: >
    certutil -d {{ firefox_profile.path }}
            -A -t "{{ ca_cert_trust_flags }}"
            -n "{{ ca_cert_name }}"
            -i /tmp/{{ ca_cert_name | replace(' ', '_') }}.crt
  loop: "{{ firefox_profiles }}"
  loop_control:
    loop_var: firefox_profile
  when: install_firefox | bool
  become: yes
  become_user: "{{ firefox_profile.user }}"

- name: Очистка временного файла сертификата
  file:
    path: /tmp/{{ ca_cert_name | replace(' ', '_') }}.crt
    state: absent
  become: yes
```

---

### 📄 `vars/main.yml`

```yaml
---
# Получаем информацию о пользователях
ca_cert_users_info: "{{ q('community.general.getent', 'passwd') | selectattr('0', 'in', ca_cert_users) | map(attribute='0') | map('community.general.user_info') | list }}"

# Собираем профили Firefox
firefox_profiles: |
  {%- set profiles = [] -%}
  {%- for user in ca_cert_users_info -%}
    {%- if user.home and (user.home + '/.mozilla/firefox') is exists -%}
      {%- for entry in q('ansible.builtin.fileglob', user.home + '/.mozilla/firefox/*.default*') -%}
        {%- if entry is directory -%}
          {{ profiles.append({'user': user.name, 'path': entry}) or '' }}
        {%- endif -%}
      {%- endfor -%}
      {%- for entry in q('ansible.builtin.fileglob', user.home + '/.mozilla/firefox/*.release') -%}
        {%- if entry is directory -%}
          {{ profiles.append({'user': user.name, 'path': entry}) or '' }}
        {%- endif -%}
      {%- endfor -%}
    {%- endif -%}
  {%- endfor -%}
  {{ profiles }}
```

> ⚠️ **Примечание**: `fileglob` и `getent` требуют установленных коллекций:
> ```bash
> ansible-galaxy collection install community.general
> ```

---

### 📌 Использование роли

#### Пример playbook:

```yaml
---
- hosts: ubuntu_servers
  become: yes
  roles:
    - role: install_ca_cert
      ca_cert_src: "/path/on/control/node/company-ca.crt"
      ca_cert_name: "Company Internal CA"
      ca_cert_users:
        - alice
        - bob
        - ubuntu
      install_chrome: true
      install_firefox: true
```

#### Или через `include_role`:

```yaml
- name: Установить CA в браузеры
  include_role:
    name: install_ca_cert
  vars:
    ca_cert_src: "{{ playbook_dir }}/files/internal-ca.crt"
    ca_cert_name: "Internal CA"
    ca_cert_users: ["{{ ansible_user }}"]
```

---

### ✅ Особенности

- Автоматически обнаруживает профили Firefox (`*.default`, `*.default-release`, `*.release`).
- Работает с любым количеством пользователей.
- Не дублирует сертификат, если он уже есть (благодаря `creates` в Chrome и идемпотентности `certutil`).
- Поддерживает как `sql:` (новый формат NSS), так и старый (если профиль использует `cert8.db` — `certutil` сам разберётся).

---

### 🔒 Безопасность

- Сертификат копируется во временный файл с правами `0644`, затем удаляется.
- Все операции выполняются от имени целевого пользователя (`become_user`), а не root (кроме установки пакета).

---

Если нужно — можно расширить поддержку Chromium, Brave и других браузеров на базе Chromium (они используют тот же `~/.pki/nssdb`).

Готово! 🚀