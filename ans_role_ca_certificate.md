Чтобы установить сертификаты (например, .cry или .cer) на сервера под управлением Ubuntu 22.04, Ubuntu 24.04 и Oracle Linux 9 с использованием Ansible роли, вы можете создать роль, которая копирует сертификаты в нужные директории и обновляет хранилище доверенных сертификатов. Вот как это можно сделать:

1. Структура роли Ansible

Создайте структуру роли, например, ca_certificate:

```
roles/ca_certificate/
├── tasks
│   └── main.yml
├── files
│   └── (здесь разместите ваши сертификаты .cry или .cer)
└── vars
    └── main.yml
```

2. Задачи роли (tasks/main.yml)

Роль будет выполнять следующие задачи:

· Копирование сертификатов в нужные директории в зависимости от ОС.
· Обновление хранилища доверенных сертификатов.
· Обработка особых требований для разных дистрибутивов.

Пример содержимого tasks/main.yml:

```yaml
- name: "Install CA certificates on Ubuntu 22.04/24.04 and Oracle Linux 9"
  block:
    - name: "Create certificate directory for Ubuntu"
      ansible.builtin.file:
        path: "/usr/local/share/ca-certificates/{{ item }}"
        state: directory
      loop: "{{ cert_directories }}"
      when: ansible_facts['os_family'] == "Debian"

    - name: "Copy certificates to Ubuntu directory"
      ansible.builtin.copy:
        src: "{{ item.src }}"
        dest: "/usr/local/share/ca-certificates/{{ item.dest }}.crt"
        owner: root
        group: root
        mode: 0644
      loop: "{{ certificates }}"
      when: ansible_facts['os_family'] == "Debian"

    - name: "Update CA certificates on Ubuntu"
      ansible.builtin.command:
        cmd: update-ca-certificates
      when: ansible_facts['os_family'] == "Debian"

    - name: "Create certificate directory for Oracle Linux 9"
      ansible.builtin.file:
        path: "/etc/pki/ca-trust/source/anchors"
        state: directory
      when: ansible_facts['os_family'] == "RedHat"

    - name: "Copy certificates to Oracle Linux directory"
      ansible.builtin.copy:
        src: "{{ item.src }}"
        dest: "/etc/pki/ca-trust/source/anchors/{{ item.dest }}.crt"
        owner: root
        group: root
        mode: 0644
      loop: "{{ certificates }}"
      when: ansible_facts['os_family'] == "RedHat"

    - name: "Update CA certificates on Oracle Linux"
      ansible.builtin.command:
        cmd: update-ca-trust
      when: ansible_facts['os_family'] == "RedHat"
```

3. Переменные роли (vars/main.yml)

Определите переменные для сертификатов и директорий:

```yaml
certificates:
  - src: "example_cert.cry"
    dest: "example_cert"
  - src: "another_cert.cer"
    dest: "another_cert"

cert_directories:
  - "my_certs"
```

4. Использование роли в плейбуке

Создайте плейбук, который использует эту роль:

```yaml
- hosts: all
  become: yes
  roles:
    - role: ca_certificate
```

5. Запуск плейбука

Запустите плейбук, указав нужные хосты:

```bash
ansible-playbook -i inventory your_playbook.yml
```

Важные примечания:

1. Формат сертификатов: Убедитесь, что ваши сертификаты в форматах .cry или .cer правильно обрабатываются. Если необходимо, преобразуйте их в формат .crt или .pem перед использованием.
2. Права доступа: Сертификаты должны быть доступны для чтения root и другими пользователями, но не должны быть изменяемыми обычными пользователями.
3. Проверка: После выполнения роли проверьте, что сертификаты добавлены в хранилище доверенных сертификатов (например, с помощью openssl или update-ca-certificates).
4. Обработка ошибок: Добавьте обработку ошибок, если сертификаты уже существуют или если копирование не удалось.

Этот подход позволяет унифицировать процесс установки сертификатов на разных дистрибутивах Linux с использованием Ansible.