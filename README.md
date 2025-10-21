# PostgreSQL Master-Slave Replication Playbook

Автоматическое развертывание PostgreSQL с настройкой hot standby репликации на Ubuntu 20.04 с использованием Ansible.

## 📋 Описание

Этот проект автоматизирует:
- ✅ Установку PostgreSQL на Debian/Ubuntu системы
- ✅ Настройку Master-Slave репликации (hot standby)
- ✅ Настройку WAL архивирования
- ✅ Создание пользователей репликации
- ✅ Health checks и мониторинг репликации

## 🏗️ Архитектура проекта

```
postgresql-playbook/
├── ansible.cfg                      # Конфигурация Ansible с best practices
├── site.yml                         # Главный playbook (entry point)
├── requirements.yml                 # Внешние роли и коллекции
├── inventory/
│   └── production/
│       ├── hosts.yml               # Inventory хостов
│       └── group_vars/
│           ├── all.yml             # Общие переменные
│           ├── postgres_master.yml # Переменные мастера
│           ├── postgres_slaves.yml # Переменные слейвов
│           └── all_vault.yml.example # Пример для паролей
└── roles/
    └── postgresql_replication/      # Кастомная роль для репликации
        ├── defaults/main.yml        # Defaults переменные
        ├── handlers/main.yml        # Handlers для рестартов
        ├── tasks/
        │   ├── main.yml            # Точка входа
        │   ├── master.yml          # Настройка master
        │   └── slave.yml           # Настройка slave
        └── templates/
            ├── pg_hba_replication.conf.j2
            └── recovery.conf.j2
```

## 🚀 Быстрый старт

### Требования

- Ansible >= 2.10
- Python 3.x
- Ubuntu 20.04 на целевых серверах
- SSH доступ с sudo правами

### 1. Установка зависимостей

```bash
# Установить все зависимости (роли + коллекции) одной командой
ansible-galaxy install -r requirements.yml
```

### 2. Настройка inventory

Отредактируйте `inventory/production/hosts.yml`:

```yaml
postgres_master:
  hosts:
    postgres_master:
      ansible_host: 192.168.0.207  # Ваш IP мастера
      
postgres_slaves:
  hosts:
    postgres_slave:
      ansible_host: 192.168.0.208  # Ваш IP слейва
```

### 3. Настройка переменных

Отредактируйте `inventory/production/group_vars/all.yml`:

```yaml
# Укажите IP вашего application server
postgres_connection_ip: "192.168.0.212"  # IP сервера с приложением

# Версия PostgreSQL
postgresql_version: "12"
```

### 4. Настройка паролей (ВАЖНО!)

```bash
# Создать файл с паролями из примера
cp inventory/production/group_vars/all_vault.yml.example \
   inventory/production/group_vars/all_vault.yml

# Отредактировать пароли
nano inventory/production/group_vars/all_vault.yml

# Зашифровать vault
ansible-vault encrypt inventory/production/group_vars/all_vault.yml

# Создать файл с паролем vault (добавьте в .gitignore!)
echo "your_vault_password" > .vault_pass
chmod 600 .vault_pass
```

### 5. Запуск playbook

```bash
# Полная установка
ansible-playbook site.yml --ask-vault-pass

# Или с vault password file
ansible-playbook site.yml --vault-password-file .vault_pass

# Только установка (без репликации)
ansible-playbook site.yml --tags install

# Только настройка репликации
ansible-playbook site.yml --tags replication

# Только health checks
ansible-playbook site.yml --tags healthcheck

# Dry-run (проверка без изменений)
ansible-playbook site.yml --check --diff
```

## 🏷️ Доступные Tags

| Tag | Описание |
|-----|----------|
| `install` | Установка PostgreSQL |
| `configure` | Настройка конфигурации |
| `replication` | Настройка репликации |
| `postgres_master` | Только мастер сервер |
| `postgres_slave` | Только слейв серверы |
| `healthcheck` | Проверки работоспособности |

## 🔧 Переменные

### Основные переменные (group_vars/all.yml)

| Переменная | Значение по умолчанию | Описание |
|------------|----------------------|----------|
| `postgresql_version` | `"12"` | Версия PostgreSQL |
| `postgres_users_no_log` | `true` | Скрывать пароли в логах |
| `postgres_connection_ip` | `"192.168.0.212"` | IP для подключений приложений |

### Vault переменные (group_vars/all_vault.yml)

| Переменная | Описание |
|------------|----------|
| `vault_postgres_password` | Пароль postgres пользователя |
| `vault_replica_password` | Пароль для репликации |

## 📊 Проверка репликации

После развертывания проверьте статус:

```bash
# На мастере - должен показать подключенные слейвы
sudo -u postgres psql -c "SELECT * FROM pg_stat_replication;"

# На слейве - должен вернуть 't' (true)
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
```

## 🔒 Безопасность

- ✅ Все пароли в Ansible Vault (AES256 encryption)
- ✅ `no_log` для sensitive операций
- ✅ `.vault_pass` в .gitignore
- ✅ Backup конфигов перед изменениями
- ⚠️ `host_key_checking = False` только для тестирования!

## 🐛 Troubleshooting

### Ошибка: "could not translate host name"
```bash
# Проверьте /etc/hosts на Ansible control node
# Убедитесь что IP адреса правильные
ansible all -m ping
```

### Ошибка: "psycopg2 not found"
```bash
# Убедитесь что python3-psycopg2 установлен
ansible all -m apt -a "name=python3-psycopg2 state=present"
```

### Ошибка: "no pg_hba.conf entry for host"
```bash
# Проверьте правильность postgres_connection_ip в group_vars/all.yml
# Проверьте pg_hba.conf на мастере
sudo cat /etc/postgresql/12/main/pg_hba.conf
```

### Репликация не работает
```bash
# Проверьте логи на мастере
sudo tail -f /var/log/postgresql/postgresql-12-main.log

# Проверьте статус репликации
sudo -u postgres psql -c "SELECT * FROM pg_stat_replication;"

# На слейве проверьте recovery mode
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
```

## 📚 Документация

- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [PostgreSQL Replication](https://www.postgresql.org/docs/current/high-availability.html)
- [geerlingguy.postgresql role](https://github.com/geerlingguy/ansible-role-postgresql)
- [community.postgresql collection](https://docs.ansible.com/ansible/latest/collections/community/postgresql/)

## 🎯 Структура проекта (Best Practices)

Проект следует Red Hat Ansible Best Practices:

- ✅ **Модульная структура**: роли для переиспользования
- ✅ **Разделение переменных**: `group_vars/` по группам хостов
- ✅ **YAML inventory**: вместо INI для лучшей читаемости
- ✅ **Handlers**: для оптимизации рестартов сервисов
- ✅ **Templates**: Jinja2 для динамических конфигов
- ✅ **Tags**: для гранулярного контроля выполнения
- ✅ **FQCN**: полные имена модулей (ansible.builtin.*)
- ✅ **Modern syntax**: `loop` вместо `with_items`
- ✅ **Error handling**: `block/rescue/always` для rollback
- ✅ **Security**: Ansible Vault для паролей
- ✅ **Documentation**: подробные комментарии в коде

## 📝 Дополнительные файлы

- **MIGRATION_GUIDE.md** - руководство по переходу со старой структуры
- **CHANGES_SUMMARY.md** - детальное описание всех изменений
- **postgres-playbook.yml** - старый монолитный playbook (для справки)

## 📖 TODO

- [ ] Добавить мониторинг через Prometheus
- [ ] Настроить автоматический failover с Patroni
- [ ] Добавить backup скрипты (pg_dump, pg_basebackup)
- [ ] Molecule tests для роли
- [ ] CI/CD интеграция
- [ ] Поддержка PostgreSQL 13, 14, 15

## 👤 Автор

Создано с использованием Ansible Best Practices от Red Hat

## 📄 Лицензия

MIT
