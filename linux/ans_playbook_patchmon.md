Просит простой playbook для выполнения команды автоподключения через curl. В документации PatchMon действительно подтверждается, что этот метод предназначен для интеграции с Ansible, а сам скрипт установки уже содержит встроенную проверку, предотвращающую повторную регистрацию хоста.

Вот готовый к использованию playbook с обработкой ошибок и защитой от повторного запуска:

```yaml
---
- name: Подключение серверов к PatchMon
  hosts: all
  become: yes
  vars:
    patchmon_url: "https://patchmon.nizhpharm.ru"
    patchmon_token_key: "patchmon_ae_..."  # Подставьте полный ключ из ссылки
    patchmon_token_secret: "..."           # Секрет (если требуется, см. примечание ниже)

  tasks:
    - name: Проверка наличия конфигурации агента (идемпотентность)
      ansible.builtin.stat:
        path: /etc/patchmon/config.yml
      register: patchmon_config

    - name: Загрузка и запуск скрипта автоподключения PatchMon
      ansible.builtin.shell: |
        curl -s "{{ patchmon_url }}/api/v1/auto-enrollment/script?type=direct-host&token_key={{ patchmon_token_key }}&token_secret={{ patchmon_token_secret }}" | sh
      args:
        executable: /bin/bash
      when: not patchmon_config.stat.exists
      register: enrollment_result

    - name: Вывод результата подключения
      ansible.builtin.debug:
        var: enrollment_result.stdout_lines
      when: enrollment_result.changed

    - name: Проверка успешности подключения
      ansible.builtin.fail:
        msg: "Ошибка подключения к PatchMon: {{ enrollment_result.stderr | default('код возврата ' ~ enrollment_result.rc) }}"
      when: enrollment_result.rc != 0
```

💡 На что обратить внимание

· Токен и секрет: На скриншоте в ссылке виден только token_key (начинается с patchmon_ae_). Если при запуске скрипт вернёт ошибку 401 Unauthorized, добавьте в переменную patchmon_token_secret секретное значение токена (его можно найти в настройках интеграций PatchMon).
· Идемпотентность: Задача пропускается, если файл /etc/patchmon/config.yml уже существует. Это защищает от повторного выполнения скрипта на уже подключённых серверах.
· Права: Playbook использует become: yes, так как установка агента и запись в /etc требуют прав суперпользователя.

Если потребуется добавить в playbook массовое развёртывание через динамический инвентарь или настроить автоматическую установку агента без curl, дайте знать.