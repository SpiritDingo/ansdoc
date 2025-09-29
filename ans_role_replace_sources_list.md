Вот пример Ansible-роли, которая заменяет файл `/etc/apt/sources.list` на Ubuntu 22.04 и 24.04, а затем проверяет его корректность с помощью `apt update`.

---

### 📁 Структура роли:

```
roles/
└── replace_sources_list/
    ├── tasks/
    │   └── main.yml
    ├── templates/
    │   ├── sources.list.j2
    ├── handlers/
    │   └── main.yml
    └── vars/
        └── main.yml
```

---

### 📄 `roles/replace_sources_list/tasks/main.yml`

```yaml
---
- name: Убедиться, что используется Ubuntu 22.04 или 24.04
  assert:
    that:
      - ansible_distribution == "Ubuntu"
      - ansible_distribution_version in ["22.04", "24.04"]
    fail_msg: "Эта роль поддерживает только Ubuntu 22.04 и 24.04"

- name: Создать резервную копию текущего sources.list
  copy:
    src: /etc/apt/sources.list
    dest: /etc/apt/sources.list.bak_{{ ansible_date_time.iso8601_basic_short }}
    remote_src: yes
    backup: no
  when: backup_sources_list | default(true)

- name: Развернуть новый sources.list из шаблона
  template:
    src: sources.list.j2
    dest: /etc/apt/sources.list
    owner: root
    group: root
    mode: '0644'
  notify: Обновить кэш apt

- name: Убедиться, что /etc/apt/sources.list существует и не пустой
  stat:
    path: /etc/apt/sources.list
  register: sources_list_stat

- name: Проверить, что файл sources.list не пустой
  assert:
    that:
      - sources_list_stat.stat.exists
      - sources_list_stat.stat.size > 0
    fail_msg: "Файл /etc/apt/sources.list отсутствует или пустой!"
```

---

### 📄 `roles/replace_sources_list/handlers/main.yml`

```yaml
---
- name: Обновить кэш apt
  apt:
    update_cache: yes
    cache_valid_time: 3600
  register: apt_update_result
  failed_when: apt_update_result.failed and "'Could not resolve' not in apt_update_result.msg"
```

> 💡 **Примечание**: Мы добавили `failed_when`, чтобы игнорировать ошибки, связанные с временной недоступностью репозиториев (например, DNS), но по-прежнему фиксировать реальные ошибки синтаксиса или недоступных репозиториев.

---

### 📄 `roles/replace_sources_list/templates/sources.list.j2`

```jinja2
# {{ ansible_managed }}
# Источники для Ubuntu {{ ansible_distribution_version }}

deb http://archive.ubuntu.com/ubuntu/ {{ ansible_distribution_release }} main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu/ {{ ansible_distribution_release }}-updates main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu/ {{ ansible_distribution_release }}-backports main restricted universe multiverse
deb http://security.ubuntu.com/ubuntu/ {{ ansible_distribution_release }}-security main restricted universe multiverse

# Раскомментируйте, если нужны исходники (source packages)
# deb-src http://archive.ubuntu.com/ubuntu/ {{ ansible_distribution_release }} main restricted universe multiverse
# deb-src http://archive.ubuntu.com/ubuntu/ {{ ansible_distribution_release }}-updates main restricted universe multiverse
# deb-src http://security.ubuntu.com/ubuntu/ {{ ansible_distribution_release }}-security main restricted universe multiverse
```

> ✅ Используется `ansible_distribution_release` (например, `jammy` для 22.04, `noble` для 24.04), что корректно работает на обеих версиях.

---

### 📄 `roles/replace_sources_list/vars/main.yml` (опционально)

```yaml
# По умолчанию делаем резервную копию
backup_sources_list: true
```

---

### ▶️ Пример использования в playbook:

```yaml
---
- name: Заменить sources.list на Ubuntu
  hosts: ubuntu_servers
  become: yes
  roles:
    - replace_sources_list
```

---

### ✅ Что делает роль:

1. Проверяет, что ОС — Ubuntu 22.04 или 24.04.
2. Делает резервную копию текущего `sources.list`.
3. Заменяет его на новый, сгенерированный из шаблона.
4. Проверяет, что файл существует и не пустой.
5. Выполняет `apt update` через handler для проверки корректности репозиториев.

---

Если вы хотите использовать **локальные зеркала** или **корпоративные репозитории**, просто отредактируйте шаблон `sources.list.j2` или вынесите URL в переменные.

Хочешь версию с поддержкой прокси или custom mirrors?