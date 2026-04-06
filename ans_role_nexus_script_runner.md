Вот Ansible роль, которая скачивает sh-скрипт с сервера Nexus и запускает его на удалённой машине. Код написан так, чтобы его можно было сразу использовать, и включает в себя безопасную работу с данными для авторизации.

📁 Структура роли

```text
nexus_script_runner/          # Корневая директория роли
├── tasks/                    # Содержит основные задачи
│   └── main.yml              # Главный файл с последовательностью задач
├── defaults/                 # Переменные по умолчанию
│   └── main.yml              # Можно определить путь для сохранения скрипта
├── vars/                     # Переменные (рекомендуется для sensitive data)
│   └── main.yml              # Переменные, которые стоит зашифровать через Ansible Vault
└── README.md                 # Документация роли
```

⚙️ Код роли (tasks/main.yml)

```yaml
---
# 1. СКАЧИВАНИЕ SCRIPT-ФАЙЛА ИЗ NEXUS
- name: "Download shell script from Nexus Repository"
  ansible.builtin.get_url:
    url: "{{ nexus_script_url }}"                  # URL на файл в Nexus (см. Примечания)
    dest: "{{ script_dest_path }}"                # Путь сохранения на целевом сервере
    url_username: "{{ nexus_username }}"          # Username для Basic Auth
    url_password: "{{ nexus_password }}"          # Password для Basic Auth
    force_basic_auth: yes                         # Отправлять credentials при первом же запросе
    mode: '0755'                                  # Права на исполнение (+x)
    backup: yes                                   # Создаст бэкап, если файл с таким именем уже существует
  register: download_result                      # Сохраняем результат задачи в переменную

# 2. ЗАПУСК СКАЧАННОГО SCRIPT-ФАЙЛА НА УДАЛЁННОМ СЕРВЕРЕ
- name: "Execute the downloaded script on the remote server"
  ansible.builtin.shell:
    cmd: "{{ script_dest_path }} {{ script_arguments | default('') }}"
    executable: /bin/bash                          # Явно укажите bash или sh, если нужен другой интерпретатор
  register: script_execution
  environment: "{{ script_environment | default({}) }}"  # Проброс переменных окружения, если нужны

# 3. ВЫВОД РЕЗУЛЬТАТА ВЫПОЛНЕНИЯ СКРИПТА
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
```

🔐 Переменные (рекомендации)

Эти данные нужно определить на уровне плейбука или группы хостов. Никогда не храните пароли в открытом виде! Используйте Ansible Vault для шифрования.

```yaml
# Рекомендуется определить в 'group_vars/all/vault.yml' и зашифровать через Ansible Vault
nexus_username: "your_username"
nexus_password: "your_password"

# Рекомендуется определить в 'group_vars/all/vars.yml'
nexus_script_url: "https://nexus.your-company.com/repository/scripts/my_script.sh"
script_dest_path: "/tmp/deployment_script.sh"

# Опционально: аргументы для скрипта
script_arguments: "--verbose --config /etc/app.conf"

# Опционально: переменные окружения для скрипта
script_environment:
  APP_ENV: "production"
  LOG_LEVEL: "INFO"
```

▶️ Использование в плейбуке

```yaml
- name: "Deploy and run Nexus script"
  hosts: your_target_hosts
  become: yes
  roles:
    - nexus_script_runner
```

📌 Ключевые моменты и пояснения

· Скачивание: Используется модуль ansible.builtin.get_url. Параметры url_username и url_password отвечают за базовую HTTP-аутентификацию. Флаг force_basic_auth: yes решает проблемы с редиректами. Также автоматически задаются права на исполнение (mode: '0755').
· Запуск: Используется модуль ansible.builtin.shell, который выполняет скрипт через стандартную командную оболочку на целевом хосте. Результат сохраняется в переменную script_execution.
· Вывод: Данные выводятся с помощью ansible.builtin.debug. Переменная script_execution.stdout_lines содержит стандартный вывод, а script_execution.stderr_lines — вывод ошибок.
· Безопасность: Обязательно зашифруйте nexus_password с помощью ansible-vault encrypt_string, чтобы избежать утечки секретов.

Если потребуется адаптировать роль под другие задачи или возникнут вопросы, обращайтесь.