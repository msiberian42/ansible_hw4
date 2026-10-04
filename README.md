# Ansible Playbook

Проект разворачивает ClickHouse, Vector и LightHouse с помощью Ansible. В `playbook/site.yml` определены три play для групп inventory `clickhouse`, `vector` и `lighthouse`. Каждый play использует соответствующую роль из GitHub; источники и версии ролей указаны в `playbook/requirements.yml`.

## Структура

```text
.
└── playbook/
    ├── site.yml
    ├── requirements.yml
    ├── inventory/
    │   └── prod.yml
    ├── group_vars/
    │   ├── clickhouse/vars.yml
    │   ├── lighthouse/vars.yml
    │   └── vector/vars.yml
    └── templates/
        ├── lighthouse.conf.j2
        ├── vector.service.j2
        ├── vector.toml.j2
        └── (прочие шаблоны)
```

## Play и роли

| Группа inventory | Роль | Репозиторий и версия |
| --- | --- | --- |
| `clickhouse` | `clickhouse` | [AlexeySetevoi/ansible-clickhouse](https://github.com/AlexeySetevoi/ansible-clickhouse), `1.13` |
| `vector` | `vector` | [msiberian42/vector-role](https://github.com/msiberian42/vector-role), `1.0` |
| `lighthouse` | `lighthouse` | [msiberian42/lighthouse-role](https://github.com/msiberian42/lighthouse-role), `1.0` |

Play для ClickHouse после выполнения роли также создаёт базу данных `logs`. Play Vector использует значения версии, архитектуры и каталогов из `group_vars/vector/vars.yml`. Play LightHouse использует параметры из `group_vars/lighthouse/vars.yml`.

Vector настроен на чтение событий из `journald` и вывод в `console`. LightHouse публикуется через Nginx по HTTP.

## Переменные

Основные переменные находятся в `playbook/group_vars/`:

- ClickHouse: `clickhouse_repo_key` (и закомментированные примеры версии и пакетов) — `clickhouse/vars.yml`.
- Vector: `vector_version`, `vector_arch`, `vector_install_dir`, `vector_config_dir` — `vector/vars.yml`.
- LightHouse: `lighthouse_version`, `lighthouse_install_dir`, `lighthouse_archive`, `lighthouse_repo` — `lighthouse/vars.yml`.

В `playbook/inventory/prod.yml` для каждого хоста задаются `ansible_host`, `ansible_user` и `ansible_python_interpreter`.

## Запуск

Из корня репозитория установите роли из GitHub в `playbook/roles`:

```bash
ansible-galaxy install -r playbook/requirements.yml -p playbook/roles
```

Затем запустите три play:

```bash
ansible-playbook -i playbook/inventory/prod.yml playbook/site.yml
```

Нужен SSH-доступ к хостам inventory и возможность повышения привилегий (`become`).

## Проверка после установки

Статус сервисов:

```bash
ansible -i playbook/inventory/prod.yml clickhouse -m ansible.builtin.command -a "systemctl is-active clickhouse-server"
ansible -i playbook/inventory/prod.yml vector -m ansible.builtin.command -a "systemctl is-active vector"
ansible -i playbook/inventory/prod.yml lighthouse -m ansible.builtin.command -a "systemctl is-active nginx"
```

Дополнительные проверки:

```bash
ansible -i playbook/inventory/prod.yml clickhouse -m ansible.builtin.command -a "clickhouse-client -q 'SHOW DATABASES'"
ansible -i playbook/inventory/prod.yml vector -m ansible.builtin.command -a "sudo /usr/local/bin/vector validate /etc/vector/vector.toml"
ansible -i playbook/inventory/prod.yml lighthouse -m ansible.builtin.command -a "nginx -t"
```
