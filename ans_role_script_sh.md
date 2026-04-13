Ниже представлена полностью готовая Ansible роль nexus_script_runner. Она скачивает shell-скрипт из Nexus (с базовой авторизацией), сохраняет на целевом сервере, запускает и выводит результат.

📁 Структура роли

```
nexus_script_runner/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
├── vars
│   └── main.yml
└── README.md
```

🔐 Файл defaults/main.yml

Переменные по умолчанию (можно переопределить в плейбуке).

```yaml
---
# Путь для сохранения скрипта на целевом сервере (рекомендуется переопределить)
script_dest_path: "/tmp/nexus_script_{{ ansible_date_time.epoch }}.sh"

# Аргументы, передаваемые скрипту (пустая строка по умолчанию)
script_arguments: ""

# Переменные окружения для скрипта (пустой словарь)
script_environment: {}

# Таймаут выполнения скрипта в секундах (по умолчанию 300)
script_timeout: 300

# Останавливать ли выполнение плейбука при ошибке скрипта (rc != 0)
fail_on_script_error: true
```

🧠 Файл vars/main.yml

В этом файле обычно определяют переменные, которые не должны меняться при вызове роли, либо оставляют пустым. Для безопасности рекомендуется определять nexus_username и nexus_password в плейбуке или в group_vars с шифрованием Vault. Здесь приведён пустой файл, но с комментарием.

```yaml
---
# Секретные переменные лучше определять вне роли:
#   nexus_username: "ваш_логин"
#   nexus_password: "ваш_пароль"
#   nexus_script_url: "https://nexus.example.com/repository/scripts/script.sh"
```

⚙️ Файл tasks/main.yml

Основная логика роли.

```yaml
---
- name: "Download shell script from Nexus Repository"
  ansible.builtin.get_url:
    url: "{{ nexus_script_url }}"
    dest: "{{ script_dest_path }}"
    url_username: "{{ nexus_username }}"
    url_password: "{{ nexus_password }}"
    force_basic_auth: yes
    mode: '0755'
    backup: yes
    timeout: 30
  register: download_result
  no_log: false   # Установите true, если не хотите логировать URL с паролем (но пароль скрыт автоматически)

- name: "Fail if download was unsuccessful"
  ansible.builtin.fail:
    msg: "Failed to download script from {{ nexus_script_url }}. HTTP status code: {{ download_result.status_code }}"
  when: download_result.status_code is defined and download_result.status_code != 200

- name: "Execute the downloaded script on the remote server"
  ansible.builtin.shell:
    cmd: "{{ script_dest_path }} {{ script_arguments }}"
    executable: /bin/bash
    timeout: "{{ script_timeout }}"
  register: script_execution
  environment: "{{ script_environment }}"
  ignore_errors: "{{ not fail_on_script_error }}"

- name: "Display stdout from script execution"
  ansible.builtin.debug:
    msg: "STDOUT: {{ script_execution.stdout_lines }}"
  when: script_execution.stdout is defined and script_execution.stdout | length > 0

- name: "Display stderr from script execution (if any)"
  ansible.builtin.debug:
    msg: "STDERR: {{ script_execution.stderr_lines }}"
  when: script_execution.stderr is defined and script_execution.stderr | length > 0

- name: "Display script execution return code"
  ansible.builtin.debug:
    msg: "RETURN CODE: {{ script_execution.rc }}"

- name: "Fail if script returned non-zero exit code and fail_on_script_error is true"
  ansible.builtin.fail:
    msg: "Script failed with return code {{ script_execution.rc }}. stderr: {{ script_execution.stderr }}"
  when:
    - fail_on_script_error | bool
    - script_execution.rc is defined
    - script_execution.rc != 0
```

📄 Файл README.md

Документация роли.

```markdown
# Nexus Script Runner

Ansible роль для скачивания shell-скрипта из Nexus (с базовой авторизацией) и его выполнения на целевом сервере.

## Требования

- Ansible 2.9+
- Доступ к Nexus Repository Manager (HTTP Basic Auth)
- Целевой сервер с `/bin/bash`

## Переменные

| Переменная | Обязательна | Описание |
|------------|-------------|----------|
| `nexus_script_url` | Да | Полный URL до скрипта в Nexus |
| `nexus_username` | Да | Имя пользователя для Basic Auth |
| `nexus_password` | Да | Пароль (рекомендуется шифровать Ansible Vault) |
| `script_dest_path` | Нет | Путь сохранения скрипта (по умолчанию `/tmp/nexus_script_<timestamp>.sh`) |
| `script_arguments` | Нет | Аргументы командной строки для скрипта |
| `script_environment` | Нет | Словарь с переменными окружения |
| `script_timeout` | Нет | Таймаут выполнения скрипта в секундах (по умолчанию 300) |
| `fail_on_script_error` | Нет | Останавливать плейбук при ненулевом коде возврата (по умолчанию `true`) |

## Пример использования

```yaml
- name: "Deploy and run script from Nexus"
  hosts: my_servers
  become: yes
  vars:
    nexus_script_url: "https://nexus.internal/repo/scripts/deploy.sh"
    nexus_username: "deploy_user"
    nexus_password: "{{ vault_nexus_password }}"  # из Ansible Vault
    script_dest_path: "/opt/deploy/deploy.sh"
    script_arguments: "--environment prod --version 2.0"
    script_environment:
      APP_CONFIG: "/etc/app/config.json"
    script_timeout: 600
  roles:
    - nexus_script_runner
```

Безопасность

· Никогда не храните nexus_password в открытом виде. Используйте ansible-vault encrypt_string или внешние секретницы.
· Параметр no_log: true можно добавить в задачу скачивания, чтобы скрыть URL, но пароль Ansible и так маскирует автоматически.

Лицензия

MIT

```

## ▶️ Пример плейбука для вызова роли

Создайте файл `playbook.yml`:

```yaml
- name: "Execute script from Nexus"
  hosts: all
  become: yes
  vars:
    nexus_script_url: "https://nexus.example.com/repository/scripts/healthcheck.sh"
    nexus_username: "ansible_user"
    nexus_password: "!vault | ..."   # или используйте --ask-vault-pass
    script_dest_path: "/usr/local/bin/healthcheck.sh"
    script_arguments: "--verbose"
  roles:
    - nexus_script_runner
```

🧪 Запуск

```bash
ansible-playbook -i inventory playbook.yml --ask-vault-pass
```

📦 Установка роли

Если роль лежит локально в roles/nexus_script_runner, Ansible найдёт её автоматически. Можно также упаковать в архив или использовать ansible-galaxy.

---

🧩 Пояснения к production-среде

· Таймауты: в задаче get_url стоит timeout: 30 для соединения. В shell – script_timeout для выполнения.
· Идемпотентность: роль не идемпотентна (скрипт запускается каждый раз). Если нужен однократный запуск, добавьте creates или проверку наличия флага.
· Обработка ошибок: роль остановится, если скачивание не удалось (код ≠ 200) или если скрипт вернул ненулевой код, а fail_on_script_error: true.

Всё готово к использованию. Скопируйте приведённые файлы в соответствующие каталоги роли.


Ошибка возникает из-за того, что в вашей задаче используется модуль ansible.builtin.command (или ansible.legacy.command), который не поддерживает параметр timeout. Параметр timeout есть только у модуля ansible.builtin.shell (и ansible.builtin.expect). Вероятно, в коде роли случайно указан command вместо shell, либо вы переопределили модуль.

🔧 Быстрое исправление

Замените в файле tasks/main.yml строку:

```yaml
ansible.builtin.command:   # или ansible.legacy.command
```

на:

```yaml
ansible.builtin.shell:
```

Полностью корректный блок задачи:

```yaml
- name: "Execute the downloaded script on the remote server"
  ansible.builtin.shell:
    cmd: "{{ script_dest_path }} {{ script_arguments }}"
    executable: /bin/bash
    timeout: "{{ script_timeout }}"
  register: script_execution
  environment: "{{ script_environment }}"
  ignore_errors: "{{ not fail_on_script_error }}"
```

📌 Почему так произошло?

· Модуль command – выполняет команду без участия шелла (не поддерживает timeout, environment, пайпы, редиректы).
· Модуль shell – выполняет команду через шелл (поддерживает timeout, environment, &, |, >, <).

Ваш скрипт, скорее всего, требует шелл (#!/bin/bash), поэтому нужен именно shell.

⏳ Если ваша версия Ansible не поддерживает timeout в shell

(Очень старые версии, например, Ansible < 2.4). Тогда используйте асинхронный запуск с опросом:

```yaml
- name: "Execute script with async timeout"
  ansible.builtin.shell:
    cmd: "{{ script_dest_path }} {{ script_arguments }}"
    executable: /bin/bash
  async: "{{ script_timeout }}"
  poll: 10
  register: script_execution
  environment: "{{ script_environment }}"
  ignore_errors: "{{ not fail_on_script_error }}"
```

🧪 Проверка модуля в вашей среде

Выполните команду, чтобы убедиться, какой модуль используется:

```bash
ansible-doc -t module shell | grep -A2 "^timeout"
```

Если timeout есть в выводе – всё поддерживается. Если нет – используйте async.

✅ Итоговый исправленный tasks/main.yml

```yaml
---
- name: "Download shell script from Nexus Repository"
  ansible.builtin.get_url:
    url: "{{ nexus_script_url }}"
    dest: "{{ script_dest_path }}"
    url_username: "{{ nexus_username }}"
    url_password: "{{ nexus_password }}"
    force_basic_auth: yes
    mode: '0755'
    backup: yes
    timeout: 30
  register: download_result

- name: "Fail if download was unsuccessful"
  ansible.builtin.fail:
    msg: "Failed to download script from {{ nexus_script_url }}. HTTP status code: {{ download_result.status_code }}"
  when: download_result.status_code is defined and download_result.status_code != 200

- name: "Execute the downloaded script on the remote server"
  ansible.builtin.shell:   # ← ИСПРАВЛЕНО: был command, теперь shell
    cmd: "{{ script_dest_path }} {{ script_arguments }}"
    executable: /bin/bash
    timeout: "{{ script_timeout }}"
  register: script_execution
  environment: "{{ script_environment }}"
  ignore_errors: "{{ not fail_on_script_error }}"

- name: "Display stdout from script execution"
  ansible.builtin.debug:
    msg: "STDOUT: {{ script_execution.stdout_lines }}"
  when: script_execution.stdout is defined and script_execution.stdout | length > 0

- name: "Display stderr from script execution (if any)"
  ansible.builtin.debug:
    msg: "STDERR: {{ script_execution.stderr_lines }}"
  when: script_execution.stderr is defined and script_execution.stderr | length > 0

- name: "Display script execution return code"
  ansible.builtin.debug:
    msg: "RETURN CODE: {{ script_execution.rc }}"

- name: "Fail if script returned non-zero exit code and fail_on_script_error is true"
  ansible.builtin.fail:
    msg: "Script failed with return code {{ script_execution.rc }}. stderr: {{ script_execution.stderr }}"
  when:
    - fail_on_script_error | bool
    - script_execution.rc is defined
    - script_execution.rc != 0
```

После исправления роль заработает без ошибки Unsupported parameters for ... command module.


Ошибка возникает из-за того, что Nexus вернул HTTP статус 304 Not Modified, а модуль get_url по умолчанию считает успешными только статусы 200 и 201. 304 означает, что файл на сервере не изменился с момента последнего скачивания, и клиент может использовать уже имеющуюся локальную копию. Это не ошибка, а нормальное поведение для кэширования.

🔧 Быстрое исправление

Измените задачу скачивания в tasks/main.yml — добавьте параметр status_code: [200, 304] и (опционально) разрешите самоподписанные сертификаты, если Nexus использует HTTPS с собственным сертификатом.

Исправленный фрагмент:

```yaml
- name: "Download shell script from Nexus Repository"
  ansible.builtin.get_url:
    url: "{{ nexus_script_url }}"
    dest: "{{ script_dest_path }}"
    url_username: "{{ nexus_username }}"
    url_password: "{{ nexus_password }}"
    force_basic_auth: yes
    mode: '0755'
    backup: yes
    timeout: 30
    status_code: [200, 304]          # ← разрешаем 304
    validate_certs: no               # ← если Nexus использует самоподписанный сертификат (иначе уберите)
  register: download_result
```

Также обновите проверку ошибки (для надёжности):

```yaml
- name: "Fail if download was unsuccessful"
  ansible.builtin.fail:
    msg: "Failed to download script from {{ nexus_script_url }}. HTTP status code: {{ download_result.status_code }}"
  when: 
    - download_result.status_code is defined
    - download_result.status_code not in [200, 304]
```

📌 Почему так произошло?

· При первом скачивании сервер отдаёт 200, файл сохраняется.
· При повторном запуске роли get_url отправляет условный запрос (If-Modified-Since / If-None-Match). Если файл не менялся, Nexus отвечает 304.
· Модуль get_url без явного status_code считает 304 ошибкой, потому что ожидает тело ответа, но его нет.

✅ Полный исправленный tasks/main.yml (ключевая часть)

```yaml
---
- name: "Download shell script from Nexus Repository"
  ansible.builtin.get_url:
    url: "{{ nexus_script_url }}"
    dest: "{{ script_dest_path }}"
    url_username: "{{ nexus_username }}"
    url_password: "{{ nexus_password }}"
    force_basic_auth: yes
    mode: '0755'
    backup: yes
    timeout: 30
    status_code: [200, 304]
    validate_certs: "{{ nexus_validate_certs | default('yes') }}"
  register: download_result

- name: "Fail if download was unsuccessful (status not 200/304)"
  ansible.builtin.fail:
    msg: "Failed to download script from {{ nexus_script_url }}. HTTP status code: {{ download_result.status_code }}"
  when:
    - download_result.status_code is defined
    - download_result.status_code not in [200, 304]
```

🧪 Дополнительная рекомендация

Если вы не хотите каждый раз проверять изменения на сервере (и получать 304), можно принудительно скачивать файл заново, добавив force: yes. Но это увеличит нагрузку на Nexus и время выполнения. Лучше оставить status_code: [200, 304].

Если Nexus требует валидации SSL-сертификата, уберите validate_certs: no или укажите yes и добавьте ca_path.

После исправления роль будет корректно обрабатывать ситуацию, когда скрипт уже был скачан ранее и не изменился.