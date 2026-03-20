Ниже представлена структура и содержимое Ansible-роли package_updater, которая умеет обновлять пакеты на Oracle Linux (RedHat-семейство) и Ubuntu (Debian-семейство). Роль поддерживает как обновление всех установленных пакетов, так и выборочное обновление конкретного пакета.

Структура роли

```
package_updater/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
└── README.md (опционально)
```

Файл defaults/main.yml

Здесь задаются значения по умолчанию. Если нужно обновить конкретный пакет, его имя передаётся через переменную package_name.

```yaml
---
# Имя пакета для выборочного обновления.
# Если не определено, обновляются все пакеты.
package_name: null
```

Файл tasks/main.yml

Основная логика: обновление кэша (только для Debian) и обновление пакетов через соответствующий модуль.

```yaml
---
- name: Update package cache (Debian/Ubuntu)
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
  when: ansible_os_family == "Debian"

- name: Upgrade all packages (Debian/Ubuntu)
  ansible.builtin.apt:
    upgrade: dist
  when:
    - ansible_os_family == "Debian"
    - package_name is not defined

- name: Upgrade all packages (RedHat/Oracle Linux)
  ansible.builtin.yum:
    name: '*'
    state: latest
  when:
    - ansible_os_family == "RedHat"
    - package_name is not defined

- name: Upgrade specific package (Debian/Ubuntu)
  ansible.builtin.apt:
    name: "{{ package_name }}"
    state: latest
    update_cache: true
  when:
    - ansible_os_family == "Debian"
    - package_name is defined

- name: Upgrade specific package (RedHat/Oracle Linux)
  ansible.builtin.yum:
    name: "{{ package_name }}"
    state: latest
  when:
    - ansible_os_family == "RedHat"
    - package_name is defined
```

Пояснения

1. Определение семейства ОС – через ansible_os_family. Для Ubuntu это Debian, для Oracle Linux – RedHat.
2. Обновление кэша выполняется только на Debian-системах (apt). Кэш считается актуальным в течение 3600 секунд, чтобы не делать лишних обращений к репозиториям.
3. Обновление всех пакетов – для Debian используется upgrade: dist (аналог apt dist-upgrade), для RedHat – yum с name: '*' и state: latest.
4. Обновление конкретного пакета – если задана переменная package_name, роль выполнит только его обновление. Для Debian перед этим дополнительно обновляется кэш.

Использование

Пример плейбука:

```yaml
- hosts: all
  roles:
    - package_updater
```

Если нужно обновить только конкретный пакет, передайте переменную:

```yaml
- hosts: all
  roles:
    - role: package_updater
      vars:
        package_name: nginx
```

Альтернативы

· Вместо yum можно использовать dnf для Oracle Linux 8+, но модуль yum совместим с обеими версиями.
· Модуль package не подходит для полного обновления системы на Debian, так как не умеет обновлять кэш. Поэтому используется раздельная логика.

Роль готова к использованию и может быть легко расширена (например, добавлена поддержка zypper для SUSE).



Дополним роль, чтобы она умела обновлять несколько пакетов, а также выводила результат обновления.

Обновлённая структура роли

```
package_updater/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
└── README.md
```

Файл defaults/main.yml

Добавляем переменную packages_to_update для списка пакетов. Старая переменная package_name остаётся для обратной совместимости, но если задан список, он имеет приоритет.

```yaml
---
# Имя одного пакета для выборочного обновления (устаревший способ, лучше использовать packages_to_update)
package_name: null

# Список пакетов для обновления (если задан, обновляются только указанные пакеты)
packages_to_update: []
```

Файл tasks/main.yml

Теперь логика учитывает список, один пакет или обновление всех. После обновления выводится детальная информация о том, какие пакеты были обновлены.

```yaml
---
# Определяем, что именно будем обновлять
- name: Determine upgrade targets
  ansible.builtin.set_fact:
    upgrade_targets: >-
      {{
        packages_to_update if packages_to_update | length > 0
        else [package_name] if package_name is defined and package_name
        else 'all'
      }}

# Обновление кэша для Debian/Ubuntu (только если есть что обновлять)
- name: Update package cache (Debian/Ubuntu)
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
  when:
    - ansible_os_family == "Debian"
    - upgrade_targets != []

# Блок обновления для Debian/Ubuntu
- name: Upgrade packages on Debian/Ubuntu
  block:
    - name: Upgrade all packages (Debian/Ubuntu)
      ansible.builtin.apt:
        upgrade: dist
      register: upgrade_result
      when: upgrade_targets == 'all'

    - name: Upgrade specific packages (Debian/Ubuntu)
      ansible.builtin.apt:
        name: "{{ upgrade_targets }}"
        state: latest
      register: upgrade_result
      when: upgrade_targets != 'all'
  when: ansible_os_family == "Debian"

# Блок обновления для RedHat/Oracle Linux
- name: Upgrade packages on RedHat/Oracle Linux
  block:
    - name: Upgrade all packages (RedHat/Oracle Linux)
      ansible.builtin.yum:
        name: '*'
        state: latest
      register: upgrade_result
      when: upgrade_targets == 'all'

    - name: Upgrade specific packages (RedHat/Oracle Linux)
      ansible.builtin.yum:
        name: "{{ upgrade_targets }}"
        state: latest
      register: upgrade_result
      when: upgrade_targets != 'all'
  when: ansible_os_family == "RedHat"

# Вывод результата обновления
- name: Show upgrade results
  ansible.builtin.debug:
    msg: |
      {% if upgrade_result.changed %}
      Обновлены пакеты:
      {% for item in upgrade_result.results %}
        - {{ item }}
      {% endfor %}
      {% else %}
      Обновление не потребовалось (все пакеты уже актуальны).
      {% endif %}
  when: upgrade_result is defined
```

Пояснения

1. Определение целей обновления – если packages_to_update не пуст, используется он; иначе если задан package_name – обновляется один пакет; иначе обновляются все пакеты.
2. Обновление кэша на Debian выполняется один раз перед операциями обновления, если есть что обновлять.
3. Два блока – для Debian и RedHat. Внутри каждого блока условие проверяет, обновляем ли мы все пакеты или конкретные. Для конкретных пакетов в name передаётся либо строка (один пакет), либо список (несколько пакетов) – модули apt и yum корректно обрабатывают оба варианта.
4. Результат сохраняется в переменную upgrade_result. Модули возвращают детальную информацию: для apt в upgrade_result.results будет список изменённых пакетов; для yum – структура аналогична. После обновления выводится читаемый отчёт.

Примеры использования

Обновление всех пакетов

```yaml
- hosts: all
  roles:
    - package_updater
```

Обновление одного пакета (старый способ)

```yaml
- hosts: all
  roles:
    - role: package_updater
      vars:
        package_name: nginx
```

Обновление нескольких пакетов

```yaml
- hosts: all
  roles:
    - role: package_updater
      vars:
        packages_to_update:
          - nginx
          - curl
          - openssl
```

Если указан и packages_to_update, и package_name, приоритет у списка.

Роль теперь универсальна и выводит понятный результат обновления.