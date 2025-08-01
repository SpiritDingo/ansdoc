Вот Ansible роль, которая выполняет указанные действия:

```yaml
---
# roles/remove_fireeye/tasks/main.yml
- name: Stop and disable xagt service
  ansible.builtin.service:
    name: xagt
    state: stopped
    enabled: no
  when: "'xagt' in ansible_facts.services"

- name: Remove xagt service
  ansible.builtin.package:
    name: xagt
    state: absent
  when: "'xagt' in ansible_facts.packages"

- name: Remove FireEye directory
  ansible.builtin.file:
    path: /opt/fireeye
    state: absent
```

Использование роли:

1. Создайте структуру каталогов:
```bash
mkdir -p roles/remove_fireeye/tasks
```

2. Поместите приведенный выше YAML код в `roles/remove_fireeye/tasks/main.yml`

3. Используйте роль в playbook:

```yaml
# remove_fireeye.yml
- hosts: all
  become: yes
  roles:
    - remove_fireeye
```

Запуск playbook:
```bash
ansible-playbook -i inventory_file remove_fireeye.yml
```

Примечания:
1. Роль сначала останавливает и отключает сервис xagt, если он существует
2. Затем пытается удалить пакет xagt, если он установлен
3. Наконец, рекурсивно удаляет каталог /opt/fireeye
4. Используется `become: yes` для выполнения задач с привилегиями root