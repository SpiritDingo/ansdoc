Ниже представлена готовая Ansible-роль, которая собирает информацию об установленных пакетах с серверов Ubuntu и Oracle Linux 9 и формирует единый CSV-отчёт.

Структура роли

```
roles/
└── collect_packages/
    ├── defaults/
    │   └── main.yml
    ├── tasks/
    │   └── main.yml
    └── templates/
        └── packages_report.csv.j2
```

---

1. defaults/main.yml

```yaml
---
# Путь к итоговому CSV-файлу на control-узле
report_path: "./installed_packages_report.csv"
```

---

2. tasks/main.yml

```yaml
---
- name: Собрать факты об установленных пакетах
  ansible.builtin.package_facts:
    manager: auto
  become: true
  tags: packages

- name: Создать единый CSV-отчёт на control-узле
  ansible.builtin.template:
    src: packages_report.csv.j2
    dest: "{{ report_path }}"
  delegate_to: localhost
  run_once: true
  become: false
  tags: report
```

Пояснение:

· package_facts сам определяет менеджер пакетов (apt для Ubuntu, dnf для Oracle Linux 9) и сохраняет данные в ansible_facts.packages.
· Шаблон выполняется один раз на localhost, но имеет доступ к фактам всех хостов через hostvars.

---

3. templates/packages_report.csv.j2

```jinja2
host,package,version,architecture
{% for host in ansible_play_hosts %}
{%   if hostvars[host]['ansible_facts']['packages'] is defined %}
{%     for pkg_name, pkg_list in hostvars[host]['ansible_facts']['packages'].items() %}
{%       for pkg in pkg_list %}
{{ host }},{{ pkg_name }},{{ pkg.version }},{{ pkg.arch | default(pkg.architecture | default('')) }}
{%       endfor %}
{%     endfor %}
{%   endif %}
{% endfor %}
```

Комментарии:

· Используется переменная ansible_play_hosts – список хостов текущего плейбука, на которых не было ошибок.
· Для каждого пакета может быть несколько записей (мультиархитектура, разные версии), поэтому внутри выполняется цикл по списку.
· Поле arch может называться по-разному в разных ОС, поэтому используется default().

---

4. Пример плейбука

```yaml
---
- name: Сбор информации об установленных пакетах
  hosts: all
  gather_facts: false          # не требуется для package_facts
  become: true
  roles:
    - collect_packages
  vars:
    report_path: "/tmp/installed_packages_report.csv"
```

---

5. Запуск

```bash
ansible-playbook -i inventory.ini playbook.yml
```

После выполнения в указанном месте (например, /tmp/installed_packages_report.csv) появится файл вида:

```csv
host,package,version,architecture
web01,acl,2.2.52-3,amd64
web01,bash,5.0-6ubuntu1.2,amd64
db01,curl,7.61.1-30.el9,x86_64
db01,openssl,3.0.7-27.0.1.el9_3,x86_64
...
```

---

Важные замечания

· Роль использует become: true для выполнения package_facts, так как некоторые менеджеры пакетов требуют привилегий.
· Если пакетов очень много, отчёт может быть большим – это нормально.
· CSV-поля не экранируются; обычно имена пакетов и версии не содержат запятых, но при необходимости можно добавить кавычки через дополнительные фильтры Jinja.
· Модуль package_facts доступен начиная с Ansible 2.5. Для Oracle Linux 9 требуется установленный Python и модуль dnf (обычно присутствует).

----------
----------


Ниже представлена обновлённая версия роли с поддержкой фильтрации пакетов. Вы можете задать регулярное выражение (или список выражений) для отбора только нужных пакетов.

Изменения в роли

1. defaults/main.yml

Добавлена переменная package_filter – по умолчанию пустой список (фильтрация отключена).

```yaml
---
# Путь к итоговому CSV-файлу на control-узле
report_path: "./installed_packages_report.csv"

# Фильтр по именам пакетов: список регулярных выражений (Python regex)
# Пример: ['^nginx', '^mysql', 'php']
# Пустой список означает "все пакеты"
package_filter: []
```

---

2. tasks/main.yml

Без изменений – модуль package_facts собирает все пакеты, а фильтрация происходит на этапе генерации отчёта.

```yaml
---
- name: Собрать факты об установленных пакетах
  ansible.builtin.package_facts:
    manager: auto
  become: true
  tags: packages

- name: Создать единый CSV-отчёт на control-узле
  ansible.builtin.template:
    src: packages_report.csv.j2
    dest: "{{ report_path }}"
  delegate_to: localhost
  run_once: true
  become: false
  tags: report
```

---

3. templates/packages_report.csv.j2

Шаблон теперь проверяет переменную package_filter и пропускает пакеты, не соответствующие ни одному из указанных регулярных выражений. Если список пуст – выгружаются все пакеты.

```jinja2
host,package,version,architecture
{% for host in ansible_play_hosts %}
{%   if hostvars[host]['ansible_facts']['packages'] is defined %}
{%     for pkg_name, pkg_list in hostvars[host]['ansible_facts']['packages'].items() %}
{%       set include_pkg = true %}
{%       if package_filter is defined and package_filter | length > 0 %}
{%         set include_pkg = false %}
{%         for pattern in package_filter %}
{%           if pkg_name is match(pattern) %}
{%             set include_pkg = true %}
{%           endif %}
{%         endfor %}
{%       endif %}
{%       if include_pkg %}
{%         for pkg in pkg_list %}
{{ host }},{{ pkg_name }},{{ pkg.version }},{{ pkg.arch | default(pkg.architecture) | default('') }}
{%         endfor %}
{%       endif %}
{%     endfor %}
{%   endif %}
{% endfor %}
```

Пояснения:

· pkg_name is match(pattern) использует регулярное выражение Python (поиск с начала строки). При необходимости можно заменить на pkg_name is search(pattern) для поиска в любом месте имени.
· Если package_filter не определён или пуст, включаются все пакеты.
· Если переменная задана строкой (не списком), Jinja будет итерировать по символам – поэтому передавайте список, даже если элемент один: package_filter: ['^nginx'].

---

Пример плейбука с фильтром

```yaml
---
- name: Сбор информации о выбранных пакетах
  hosts: all
  gather_facts: false
  become: true
  roles:
    - collect_packages
  vars:
    report_path: "/tmp/filtered_packages.csv"
    package_filter:
      - '^nginx'
      - '^mysql'
      - 'php'
```

Такой плейбук соберёт только пакеты, имена которых начинаются с nginx, mysql или содержат php.

---

Запуск

```bash
ansible-playbook -i inventory.ini playbook.yml
```

Результат будет записан в /tmp/filtered_packages.csv.

---

Дополнительные возможности

· Регистронезависимый поиск – замените is match(pattern) на is match(pattern, ignorecase=True).
· Поиск в любом месте имени – используйте is search(pattern) вместо is match(pattern).
· Фильтр по архитектуре или версии – можно расширить условие, добавив проверки внутри циклов по пакетам.

Если нужна фильтрация по другим параметрам (версия, архитектура) – сообщите, доработаю шаблон.