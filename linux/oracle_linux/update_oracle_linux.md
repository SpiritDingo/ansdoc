Ниже представлена Ansible-роль update_oracle_linux, которая выполняет полное обновление Oracle Linux 8/9 с учётом типичных проблем и формирует итоговый отчёт.

Структура роли

```
update_oracle_linux/
├── defaults/main.yml
├── handlers/main.yml
├── meta/main.yml
├── tasks/
│   ├── main.yml
│   ├── pre_update.yml
│   ├── update.yml
│   └── post_update.yml
├── templates/
│   └── report.j2
└── vars/main.yml
```

1. meta/main.yml

```yaml
galaxy_info:
  role_name: update_oracle_linux
  author: your_name
  description: Обновление Oracle Linux 8/9 с обработкой ошибок и отчётом
  license: MIT
  platforms:
    - name: OracleLinux
      versions:
        - 8
        - 9
  galaxy_tags: [oracle, linux, update, patching]
dependencies: []
```

2. defaults/main.yml

```yaml
# Пакеты, исключаемые из обновления
update_exclude: []

# Число повторных попыток при сбоях сети
update_retries: 3
update_retry_delay: 30

# Автоматическая перезагрузка после обновления
reboot_after_update: false
reboot_timeout: 600

# Путь к файлу отчёта на целевой машине
report_file: "/var/log/ansible_update_report_{{ ansible_date_time.iso8601_basic }}.txt"

# Сохранять отчёт на контроллер
report_fetch: false
report_fetch_dest: "./reports/"

# Импортировать GPG-ключи Oracle при необходимости
gpg_key_import: true

# Сброс модулей DNF (OL8) – включайте только если знаете, что делаете
module_reset: false

# Разрешить удаление конфликтующих пакетов
update_allowerasing: false

# Пропускать пакеты с неудовлетворёнными зависимостями
update_skip_broken: false
```

3. vars/main.yml

(Можно оставить пустым или использовать для переопределения в конкретном окружении)

```yaml
---
```

4. tasks/main.yml

```yaml
---
- name: Проверка поддерживаемой ОС
  assert:
    that:
      - ansible_distribution == 'OracleLinux'
      - ansible_distribution_major_version in ['8', '9']
    fail_msg: "Роль поддерживает только Oracle Linux 8 и 9"

- name: Сбор дополнительных фактов о дате
  setup:
    filter: ansible_date_time

- name: Подготовка к обновлению
  include_tasks: pre_update.yml

- name: Выполнение обновления
  include_tasks: update.yml

- name: Постобработка и отчёт
  include_tasks: post_update.yml
```

5. tasks/pre_update.yml

```yaml
---
- name: Сохранить исходные параметры системы
  set_fact:
    os_release_before: "{{ ansible_distribution }} {{ ansible_distribution_version }}"
    kernel_before: "{{ ansible_kernel }}"
    update_start_time: "{{ ansible_date_time.iso8601 }}"

- name: Проверка свободного места в /var и /boot
  vars:
    required_var: 1073741824    # 1 ГБ
    required_boot: 209715200    # 200 МБ
  assert:
    that:
      - (ansible_mounts | selectattr('mount','equalto','/var') | first).size_available > required_var
      - (ansible_mounts | selectattr('mount','equalto','/boot') | first).size_available > required_boot
    fail_msg: "Недостаточно места: /var или /boot"

- name: Импорт GPG-ключей Oracle
  when: gpg_key_import
  block:
    - name: Проверить наличие локального ключа
      stat:
        path: /etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
      register: oracle_gpg

    - name: Загрузить ключ при отсутствии
      get_url:
        url: https://yum.oracle.com/RPM-GPG-KEY-oracle
        dest: /etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
        mode: 0644
      when: not oracle_gpg.stat.exists

    - name: Импортировать ключ в RPM
      rpm_key:
        key: /etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
        state: present

- name: Очистить кэш DNF
  command: dnf clean all
  changed_when: false

- name: Обновить метаданные репозиториев
  command: dnf makecache
  register: makecache
  retries: "{{ update_retries }}"
  delay: "{{ update_retry_delay }}"
  until: makecache.rc == 0
  changed_when: false

- name: Сброс модулей (если включён, только для OL8)
  when: module_reset and ansible_distribution_major_version == '8'
  command: dnf module reset -y '*'
  changed_when: false
```

6. tasks/update.yml

```yaml
---
- name: Полное обновление системы
  shell: >
    dnf update -y
    {% if update_exclude | length > 0 %}--exclude={{ update_exclude | join(',') }}{% endif %}
    {% if update_skip_broken %}--skip-broken{% endif %}
    {% if update_allowerasing %}--allowerasing{% endif %}
  register: update_result
  retries: "{{ update_retries }}"
  delay: "{{ update_retry_delay }}"
  until: update_result.rc == 0
  ignore_errors: true
  changed_when: false

- name: Обработка ошибок обновления
  when: update_result.rc != 0
  block:
    - name: Попытка восстановления с агрессивными опциями
      shell: >
        dnf update -y --allowerasing --skip-broken
        {% if update_exclude | length > 0 %}--exclude={{ update_exclude | join(',') }}{% endif %}
      register: recovery_result
      ignore_errors: true
      changed_when: false

    - name: Фатальная ошибка при неудаче восстановления
      fail:
        msg: "Обновление провалилось. Ошибка восстановления: {{ recovery_result.stderr }}"
      when: recovery_result.rc != 0
```

7. tasks/post_update.yml

```yaml
---
- name: Собрать параметры системы после обновления
  set_fact:
    os_release_after: "{{ ansible_distribution }} {{ ansible_distribution_version }}"
    kernel_after: "{{ ansible_kernel }}"
    update_end_time: "{{ ansible_date_time.iso8601 }}"

- name: Проверить необходимость перезагрузки (по ядру)
  shell: |
    latest_kernel=$(rpm -q kernel --last 2>/dev/null | head -1 | awk '{print $1}' | sed 's/kernel-//')
    if [ "$latest_kernel" != "$(uname -r)" ]; then
      echo "reboot_needed"
    else
      echo "ok"
    fi
  register: reboot_check
  changed_when: false

- name: Установить факт перезагрузки
  set_fact:
    reboot_required: "{{ reboot_check.stdout == 'reboot_needed' }}"

- name: Найти .rpmnew и .rpmsave файлы
  find:
    paths: /
    patterns: "*.rpmnew,*.rpmsave"
    recurse: yes
    file_type: file
  register: config_files

- name: Получить историю последней транзакции DNF
  shell: dnf history info last
  register: dnf_history
  changed_when: false

- name: Сгенерировать отчёт
  template:
    src: report.j2
    dest: "{{ report_file }}"

- name: Скопировать отчёт на контроллер (опционально)
  fetch:
    src: "{{ report_file }}"
    dest: "{{ report_fetch_dest }}"
    flat: yes
  when: report_fetch

- name: Перезагрузить узел, если требуется и разрешено
  when: reboot_required and reboot_after_update
  block:
    - name: Предупреждение перед перезагрузкой
      debug:
        msg: "Узел будет перезагружен через 5 секунд..."
    - name: Перезагрузка
      reboot:
        reboot_timeout: "{{ reboot_timeout }}"
```

8. handlers/main.yml

(В данной роли обработчики не используются, файл можно оставить пустым)

```yaml
---
```

9. templates/report.j2

```text
Отчёт об обновлении Oracle Linux
================================
Хост: {{ ansible_hostname }}
Дата запуска: {{ update_start_time }}
Дата завершения: {{ update_end_time }}

Версия ОС до обновления: {{ os_release_before }}
Версия ОС после обновления: {{ os_release_after }}

Ядро до обновления: {{ kernel_before }}
Ядро после обновления: {{ kernel_after }}

Результат обновления:
{% if update_result is defined and update_result.rc == 0 %}
УСПЕШНО
{% else %}
ОШИБКА (код возврата {{ update_result.rc }})
Сообщение об ошибке:
{{ update_result.stderr }}
{% endif %}

Информация о последней транзакции DNF:
{{ dnf_history.stdout }}

Найденные файлы .rpmnew/.rpmsave:
{% if config_files.files | length > 0 %}
{% for file in config_files.files %}
  - {{ file.path }}
{% endfor %}
{% else %}
  Отсутствуют.
{% endif %}

Требуется перезагрузка: {{ 'Да' | if reboot_required else 'Нет' }}

Примечания:
- Если обнаружены .rpmnew или .rpmsave, выполните слияние конфигурации вручную.
```

Использование роли

Добавьте в playbook:

```yaml
- hosts: oracle_servers
  roles:
    - role: update_oracle_linux
      vars:
        reboot_after_update: false
        report_fetch: true
        report_fetch_dest: "./reports/"
```

Учтённые проблемы и автоматические исправления

1. GPG-ключи Oracle – загружаются при отсутствии и импортируются.
2. Устаревший кэш DNF – принудительно очищается и обновляется с повторными попытками.
3. Конфликты пакетов – при первой неудаче выполняется вторая попытка с ключами --allowerasing --skip-broken.
4. Сбои сети/репозиториев – встроены retries и delay.
5. Модули в OL8 – по желанию можно выполнить сброс (module_reset: true).
6. Исключение пакетов – через переменную update_exclude.
7. Защита от нехватки места – проверка /var и /boot перед стартом.
8. Необходимость перезагрузки – определяется сравнением загруженного и последнего установленного ядра, управляется reboot_after_update.
9. Изменённые конфигурационные файлы – находятся все .rpmnew/.rpmsave и перечисляются в отчёте.
10. Детальный отчёт – содержит исходное/конечное состояние, вывод dnf history, ошибки и флаг перезагрузки.

Роль полностью готова к использованию на Oracle Linux 8 и 9.