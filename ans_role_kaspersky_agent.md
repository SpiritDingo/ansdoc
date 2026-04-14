Для автоматической установки агента администрирования Kaspersky (klnagent) с помощью Ansible можно использовать готовые роли, доступные на GitHub и других платформах. Вот как это можно сделать.

🚀 Готовые роли на GitHub

1. ansible-kaspersky-role от AlexKirmasov

Это готовая роль для CentOS 7, которая включает в себя установку как агента администрирования (klnagent), так и Kaspersky Endpoint Security (KESL).

Ключевые особенности:

· Структура: Роль имеет типовую структуру, что упрощает её интеграцию в ваш проект.
· Методы установки: Предлагает два варианта:
  · autonomous_package: Установка из автономного пакета (например, klnagent64-11.0.0-38.x86_64.sh).
  · custom_install: Кастомная установка, при которой пакет загружается по указанному URL.
· Переменные: Для настройки используются следующие переменные:
  · metod_klnagent_param: Выбор метода установки (custom_install или autonomous_package).
  · klnagent_url: URL-адрес для скачивания пакета агента.
  · klnagent_autonomous_package_name: Имя файла автономного пакета.
  · KLNAGENT_SERVER: IP-адрес или имя сервера администрирования Kaspersky Security Center.
  · KLNAGENT_PORT: Порт для подключения к серверу администрирования (по умолчанию 14000).
  · KLNAGENT_SSLPORT: SSL-порт (по умолчанию 13000).
  · KLNAGENT_USESSL: Использовать ли SSL (1 — да, 0 — нет).
· Запуск: Плейбуки можно запускать как для всех хостов, так и для выборочных.
  ```bash
  # Для всех хостов
  ansible-playbook service_install_klnagent_all_servers.yml --vault-password-file=/path/to/secret --diff -vv
  
  # Для выбранных хостов
  ansible-playbook service_install_klnagent_select_servers.yml -e "host=test01" --vault-password-file=/path/to/secret --diff -vv
  ```
  Для хранения конфиденциальных данных (логинов и паролей от репозитория) используется ansible-vault.

2. Репозиторий ittxx/kaspersky

Этот репозиторий также содержит материалы по установке KESL и может быть полезен в дополнение к основной роли.

📝 Пример плейбука для установки агента

Ниже представлен упрощенный пример, который вы можете адаптировать под свои нужды.

```yaml
---
- name: Установка агента администрирования Kaspersky
  hosts: all
  become: yes
  vars:
    # Метод установки: autonomous_package или custom_install
    metod_klnagent_param: "custom_install"
    # URL для скачивания дистрибутива
    klnagent_url: "http://your-repo-server/klnagent64-11.0.0-38.x86_64.sh"
    # Настройки подключения к KSC
    KLNAGENT_SERVER: "192.168.1.100"
    KLNAGENT_PORT: "14000"
    KLNAGENT_SSLPORT: "13000"
    KLNAGENT_USESSL: "1"
    # Имя файла дистрибутива
    klnagent_file_name: "klnagent64-11.0.0-38.x86_64.sh"

  tasks:
    - name: Создать временную директорию
      file:
        path: /tmp/kaspersky_install
        state: directory
        mode: '0755'

    - name: Скачать дистрибутив агента
      get_url:
        url: "{{ klnagent_url }}"
        dest: "/tmp/kaspersky_install/{{ klnagent_file_name }}"
        mode: '0755'
      when: metod_klnagent_param == "custom_install"

    - name: Запустить установку агента
      shell: "/tmp/kaspersky_install/{{ klnagent_file_name }} --autoinstall"
      args:
        creates: /opt/kaspersky/klnagent64/sbin/klnagent
      when: metod_klnagent_param == "custom_install"
```

💡 Важные моменты

· Система-хранилище (репозиторий): Для успешной работы через custom_install необходимо развернуть HTTP-сервер (например, Nginx), который будет раздавать дистрибутивы. В официальной документации Kaspersky есть инструкции по настройке такого репозитория.
· Автоответы (autoanswers): При использовании метода autonomous_package или кастомной установки для автоматизации ответов на запросы установщика можно подготовить файл с настройками, например, autoanswers.conf, где указываются параметры подключения к серверу администрирования.
· Безопасность: Всегда следуйте принципу наименьших привилегий и храните учетные данные (логины, пароли) в зашифрованном виде с помощью ansible-vault.

Надеюсь, эта информация поможет вам в автоматизации. Если у вас появятся дополнительные вопросы, обращайтесь.