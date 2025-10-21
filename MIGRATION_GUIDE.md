# Руководство по миграции на новую структуру

Этот документ описывает переход от старого монолитного playbook к новой модульной структуре с best practices.

## 🔄 Что изменилось

### Старая структура
```
postgresql-playbook/
├── ansible.cfg              # Минимальная конфигурация
├── inventory.ini            # INI формат
├── postgres-playbook.yml    # Монолитный playbook (210 строк)
├── requirements.yml
└── README.md                # Пустой
```

### Новая структура
```
postgresql-playbook/
├── ansible.cfg              # ✅ Best practices конфигурация
├── site.yml                 # ✅ Главный playbook
├── requirements.yml
├── README.md                # ✅ Полная документация
├── MIGRATION_GUIDE.md       # ✅ Это руководство
├── inventory/
│   └── production/
│       ├── hosts.yml        # ✅ YAML inventory
│       └── group_vars/      # ✅ Переменные вынесены отдельно
└── roles/
    └── postgresql_replication/  # ✅ Кастомная роль
```

## 📝 Изменения в деталях

### 1. Inventory: INI → YAML
**Было (`inventory.ini`):**
```ini
[postgres:children]
postgres_master
postgres_slaves

[postgres_master]
postgres_master ansible_host=192.168.0.207 ...
```

**Стало (`inventory/production/hosts.yml`):**
```yaml
all:
  children:
    postgres:
      children:
        postgres_master:
          hosts:
            postgres_master:
              ansible_host: 192.168.0.207
```

**Преимущества:**
- ✅ Более читаемый формат
- ✅ Поддержка сложных структур данных
- ✅ Лучшая валидация синтаксиса

### 2. Переменные: встроенные → group_vars
**Было:**
```yaml
- name: "Setup master"
  vars:
    replica_password: "replica123"
    postgres_master_ip: "{{ hostvars['postgres_master']['ansible_host'] }}"
```

**Стало (`group_vars/postgres_master.yml`):**
```yaml
replica_password: "{{ vault_replica_password }}"
postgres_master_ip: "{{ hostvars['postgres_master']['ansible_host'] }}"
postgresql_master_config:
  - regexp: "#listen_addresses = 'localhost'"
    line: "listen_addresses = '{{ postgres_master_ip }}'"
```

**Преимущества:**
- ✅ Переменные в одном месте
- ✅ Легко переиспользовать
- ✅ Разделение по окружениям (dev/staging/production)

### 3. Таски: монолитные → модульная роль
**Было:** Все таски в одном файле на 210 строк

**Стало:**
```
roles/postgresql_replication/
├── tasks/
│   ├── main.yml      # Orchestration (10 строк)
│   ├── master.yml    # Master конфигурация (71 строка)
│   └── slave.yml     # Slave конфигурация (100 строк)
├── handlers/         # Handlers для перезапуска
├── templates/        # Jinja2 шаблоны
└── defaults/         # Defaults переменные
```

**Преимущества:**
- ✅ Модульность и переиспользование
- ✅ Легко тестировать отдельно
- ✅ Можно публиковать на Ansible Galaxy

### 4. Синтаксис: устаревший → современный

| Было | Стало | Почему |
|------|-------|--------|
| `with_items:` | `loop:` | Унифицированный синтаксис циклов |
| `systemd:` | `ansible.builtin.systemd:` | FQCN предотвращает конфликты |
| `encrypted: yes` | `encrypted: true` | Правильный boolean тип |
| `shell: mv ...` | `command: mv ...` | Безопаснее когда shell не нужен |
| Inline restart | `notify: restart postgresql` | Handlers запускаются 1 раз |

### 5. Безопасность: пароли в playbook → Ansible Vault

**Было:**
```yaml
vars:
  replica_password: "replica123"  # 😱 В открытом виде!
```

**Стало:**
```yaml
# group_vars/all_vault.yml (зашифрован)
vault_replica_password: "replica123"

# group_vars/postgres_master.yml
replica_password: "{{ vault_replica_password }}"
```

**Преимущества:**
- ✅ Пароли зашифрованы AES256
- ✅ Можно коммитить в Git безопасно
- ✅ `no_log` скрывает из логов

## 🚀 Как мигрировать

### Вариант 1: Использовать новую структуру (рекомендуется)

```bash
# 1. Старый playbook всё ещё работает
ansible-playbook postgres-playbook.yml

# 2. Для использования новой структуры:
# Создать vault файл
cp inventory/production/group_vars/all_vault.yml.example \
   inventory/production/group_vars/all_vault.yml

# Отредактировать пароли
nano inventory/production/group_vars/all_vault.yml

# Зашифровать
ansible-vault encrypt inventory/production/group_vars/all_vault.yml

# 3. Запустить новый playbook
ansible-playbook site.yml --ask-vault-pass
```

### Вариант 2: Постепенная миграция

Можно использовать оба playbook параллельно:

```bash
# Старый (работает как раньше)
ansible-playbook postgres-playbook.yml

# Новый (тестирование)
ansible-playbook site.yml --ask-vault-pass --check
```

## ⚙️ Что работает по-другому

### Запуск playbook
**Было:**
```bash
ansible-playbook postgres-playbook.yml
```

**Стало:**
```bash
# С vault password prompt
ansible-playbook site.yml --ask-vault-pass

# Или с password file
ansible-playbook site.yml --vault-password-file .vault_pass

# С tags для выборочного запуска
ansible-playbook site.yml --tags replication
```

### Inventory
**Было:**
```bash
ansible-playbook -i inventory.ini postgres-playbook.yml
```

**Стало (автоматически из ansible.cfg):**
```bash
ansible-playbook site.yml  # Использует inventory/production/
```

### Редактирование переменных
**Было:** Редактировать vars в playbook

**Стало:** Редактировать файлы в `inventory/production/group_vars/`

## 🔍 Проверка совместимости

```bash
# 1. Проверить синтаксис
ansible-playbook site.yml --syntax-check

# 2. Dry-run (ничего не меняет)
ansible-playbook site.yml --check --diff

# 3. Проверить connectivity
ansible all -m ping

# 4. Посмотреть какие переменные применятся
ansible-inventory --graph --vars
```

## 📊 Сравнение производительности

| Метрика | Старый playbook | Новый playbook |
|---------|-----------------|----------------|
| Строк кода | 210 | 71 + 100 + 10 (модульно) |
| Файлов конфигурации | 1 | 8 (специализированных) |
| Переиспользуемость | Низкая | Высокая (роль) |
| Тестируемость | Сложно | Легко (по модулям) |
| Безопасность | Пароли открыто | Vault encryption |
| Читаемость | Средняя | Высокая |

## ❓ FAQ

### Q: Нужно ли удалять старый playbook?
A: Нет, оставьте `postgres-playbook.yml` для справки или fallback. Он никак не мешает.

### Q: Как откатиться на старую версию?
A: Просто используйте старый playbook:
```bash
ansible-playbook postgres-playbook.yml -i inventory.ini
```

### Q: Нужно ли переделывать существующие серверы?
A: Нет! Новый playbook idempotent и безопасно применится к уже настроенным серверам.

### Q: Что если забыл vault password?
A: Храните его в безопасном месте (password manager). Если потеряли - придется пересоздать vault.

### Q: Можно ли использовать старый inventory.ini?
A: Да, но рекомендуется мигрировать на YAML для консистентности.

## 📚 Дополнительные ресурсы

- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [Ansible Vault Guide](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
- [Ansible Roles](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html)

## ✅ Чеклист миграции

- [ ] Прочитать README.md
- [ ] Создать и зашифровать all_vault.yml
- [ ] Проверить inventory/production/hosts.yml
- [ ] Запустить `--syntax-check`
- [ ] Запустить `--check` (dry-run)
- [ ] Протестировать на dev/staging
- [ ] Применить на production
- [ ] Проверить репликацию
- [ ] Обновить документацию команды
