# Ansible роль для проверки локализации Linux

Вот пример Ansible роли для проверки и настройки локализации в Linux-системах.

## Структура роли

```
roles/locale_check/
├── defaults
│   └── main.yml      # параметры по умолчанию
├── tasks
│   └── main.yml      # основные задачи
├── templates
│   └── locale.j2     # шаблон для генерации файла локали
└── README.md         # документация
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки локализации по умолчанию
locale_default_lang: "en_US.UTF-8"
locale_supported:
  - "en_US.UTF-8 UTF-8"
  - "ru_RU.UTF-8 UTF-8"
locale_system_lang: "en_US.UTF-8"
locale_generate: true
locale_apply: true
```

### tasks/main.yml

```yaml
---
- name: Проверить текущую локализацию
  command: locale
  register: current_locale
  changed_when: false

- name: Показать текущую локализацию
  debug:
    var: current_locale.stdout_lines

- name: Проверить доступные локали
  command: locale -a
  register: available_locales
  changed_when: false

- name: Показать доступные локали
  debug:
    var: available_locales.stdout_lines

- name: Убедиться, что пакет локалей установлен
  package:
    name: "locales"
    state: present
  when: ansible_os_family == 'Debian'

- name: Убедиться, что пакет glibc-locale-source установлен (для RHEL)
  package:
    name: "glibc-locale-source"
    state: present
  when: ansible_os_family == 'RedHat'

- name: Создать шаблон для локали
  template:
    src: locale.j2
    dest: /etc/locale.gen
    owner: root
    group: root
    mode: '0644'
  when: locale_generate

- name: Сгенерировать локали
  command: locale-gen
  when: 
    - locale_generate
    - ansible_os_family == 'Debian'

- name: Установить системную локаль
  command: localectl set-locale LANG={{ locale_system_lang }}
  when: 
    - locale_apply
    - ansible_os_family in ['RedHat', 'Suse', 'Debian']

- name: Обновить переменные локали
  block:
    - name: Создать файл /etc/default/locale (Debian)
      copy:
        dest: /etc/default/locale
        content: |
          LANG={{ locale_system_lang }}
          LC_ALL={{ locale_system_lang }}
      when: ansible_os_family == 'Debian'

    - name: Обновить /etc/locale.conf (RHEL)
      copy:
        dest: /etc/locale.conf
        content: |
          LANG={{ locale_system_lang }}
          LC_ALL={{ locale_system_lang }}
      when: ansible_os_family == 'RedHat'

    - name: Применить новые настройки локали
      command: source /etc/default/locale
      when: ansible_os_family == 'Debian'
  when: locale_apply
```

### templates/locale.j2

```
# Этот файл сгенерирован Ansible
# Список поддерживаемых локалей

{{ locale_system_lang }} UTF-8
{% for locale in locale_supported %}
{{ locale }}
{% endfor %}
```

## Использование роли

1. Создайте playbook (например, `locale_check.yml`):

```yaml
---
- hosts: all
  become: yes
  roles:
    - locale_check
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory locale_check.yml
```

## Кастомизация

Вы можете переопределить параметры по умолчанию:

```yaml
---
- hosts: all
  become: yes
  vars:
    locale_system_lang: "ru_RU.UTF-8"
    locale_supported:
      - "en_US.UTF-8 UTF-8"
      - "ru_RU.UTF-8 UTF-8"
      - "en_GB.UTF-8 UTF-8"
  roles:
    - locale_check
```

## Дополнительные возможности

Для более полной проверки вы можете добавить следующие задачи:

```yaml
- name: Проверить текущий часовой пояс
  command: timedatectl
  register: timezone_info
  changed_when: false

- name: Показать информацию о часовом поясе
  debug:
    var: timezone_info.stdout_lines

- name: Проверить кодировку терминала
  command: echo $LANG $LC_ALL
  register: terminal_encoding
  changed_when: false

- name: Показать кодировку терминала
  debug:
    var: terminal_encoding.stdout
```

Эта роль обеспечит проверку текущих настроек локализации и при необходимости их настройку в соответствии с заданными параметрами.