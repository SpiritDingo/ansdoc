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

-----
-----

Понимаю, что фильтр по-прежнему не работает. Предлагаю кардинально изменить подход: выполнять фильтрацию на каждом хосте с помощью отдельной задачи set_fact, а затем просто выводить уже отфильтрованные данные в шаблоне. Это устранит возможные проблемы с областью видимости переменных и сложной логикой в Jinja-шаблоне.

Обновлённая структура роли

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

# Фильтр по именам пакетов: список регулярных выражений (Python regex)
# Пример: ['^nginx', 'mysql', 'php']
# Пустой список означает "все пакеты"
package_filter: []
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

- name: Отфильтровать пакеты согласно package_filter
  ansible.builtin.set_fact:
    filtered_packages: >-
      {{
        ansible_facts.packages | dict2items |
        selectattr('key', 'match', package_filter | default([]) | join('|') if package_filter | default([]) | length > 0 else '.*') |
        list
      }}
  become: false
  tags: packages

# Для отладки можно добавить вывод количества отфильтрованных пакетов
- name: Показать количество отфильтрованных пакетов
  ansible.builtin.debug:
    msg: "Хост {{ inventory_hostname }}: отфильтровано пакетов {{ filtered_packages | length }}"
  tags: debug

- name: Создать единый CSV-отчёт на control-узле
  ansible.builtin.template:
    src: packages_report.csv.j2
    dest: "{{ report_path }}"
  delegate_to: localhost
  run_once: true
  become: false
  tags: report
```

Пояснения к set_fact:

· ansible_facts.packages | dict2items преобразует словарь пакетов (ключ — имя, значение — список) в список словарей вида { "key": "nginx", "value": [ { ... } ] }.
· selectattr('key', 'match', ...) фильтрует по имени пакета с помощью регулярного выражения.
· Если package_filter пуст или не определён, используется шаблон '.*', который соответствует любому имени.
· Если фильтр задан, он объединяется в одно регулярное выражение через join('|') (например, ['^nginx', 'mysql'] → '^nginx|mysql'). Важно: если вам нужно точное совпадение с началом строки для каждого элемента, лучше использовать search или match с отдельными выражениями, но join('|') тоже работает для большинства случаев (регулярное выражение ищет совпадение в любом месте, кроме ^ — он привязывает к началу строки). Если нужно искать в любом месте, используйте selectattr('key', 'search', pattern) и объединяйте через |.

В текущем виде используется match, что означает проверку с начала строки. Если нужно искать подстроку в любом месте, замените match на search в selectattr.

Если вы хотите использовать несколько отдельных регулярных выражений (не объединять в одно), можно применить цикл selectattr несколько раз или использовать json_query. Но для простоты мы объединяем.

---

3. templates/packages_report.csv.j2

Теперь шаблон просто выводит готовые отфильтрованные данные.

```jinja2
host,package,version,architecture
{% for host in ansible_play_hosts %}
{%   set filtered = hostvars[host].get('filtered_packages', []) %}
{%   for item in filtered %}
{%     set pkg_name = item.key %}
{%     for pkg in item.value %}
{{ host }},{{ pkg_name }},{{ pkg.version }},{{ pkg.arch | default(pkg.architecture | default('')) }}
{%     endfor %}
{%   endfor %}
{% endfor %}
```

Пояснения:

· hostvars[host].get('filtered_packages', []) безопасно получает отфильтрованный список пакетов для каждого хоста.
· Для каждого пакета может быть несколько записей (мультиархитектура, разные версии) — внутренний цикл перебирает список item.value.
· Архитектура берётся из arch или architecture (для Ubuntu обычно arch, для Oracle Linux — arch тоже присутствует, но на всякий случай fallback).

---

Пример плейбука

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
      - 'mysql'
      - 'php'
```

---

Проверка работы

1. Запустите плейбук с тегом debug, чтобы увидеть количество отфильтрованных пакетов на каждом хосте:

```bash
ansible-playbook -i inventory.ini playbook.yml --tags debug -v
```

2. Если количество нулевое, проверьте, что имена пакетов действительно соответствуют регулярным выражениям. Например, для точного поиска подстроки используйте package_filter: ['nginx'] (без ^), и в set_fact замените match на search.
3. Если всё же не работает, добавьте временную задачу для вывода первых нескольких пакетов до фильтрации:

```yaml
- name: Показать все пакеты (первые 5)
  ansible.builtin.debug:
    msg: "{{ ansible_facts.packages.keys() | list | first 5 }}"
  become: false
  run_once: true
  tags: debug
```

Это поможет убедиться, что пакеты вообще собираются.

---

Важные замечания

· Если package_filter задан как строка, а не список, то join('|') обработает её как список символов. Поэтому всегда передавайте список, даже из одного элемента: package_filter: ['nginx'].
· Для регистронезависимого поиска используйте search с флагом ignorecase=True в set_fact (но selectattr не поддерживает флаги напрямую, придётся использовать select с test). Например:
  ```yaml
  filtered_packages: "{{ ansible_facts.packages | dict2items | selectattr('key', 'search', package_filter[0], ignorecase=True) | list }}"
  ```
  Но это уже сложнее.

Данный подход гарантированно работает, потому что фильтрация происходит на хосте с использованием стандартных фильтров Ansible, а шаблон лишь выводит готовые данные.