# Operations Guide

## Первый деплой

```bash
# 0. Установить зависимости Ansible
ansible-galaxy collection install -r requirements.yml

# 1. Заполнить vault
cp inventory/group_vars/all/vault.yml /tmp/vault_template.yml
# отредактировать все CHANGE_ME значения
ansible-vault encrypt inventory/group_vars/all/vault.yml

# 2. Создать .vault_pass
echo "your_vault_password" > .vault_pass
chmod 600 .vault_pass

# 3. Проверить инвентарь
ansible-inventory --list

# 4. Проверить коннективность
ansible all -m ping

# 5. Деплой по шагам (первый раз лучше пошагово)
ansible-playbook playbooks/01_common.yml
ansible-playbook playbooks/02_postgres.yml
ansible-playbook playbooks/03_redis.yml
ansible-playbook playbooks/04_coturn.yml
ansible-playbook playbooks/05_livekit.yml
ansible-playbook playbooks/06_synapse.yml

# 6. Сгенерировать ключи MAS (один раз)
ansible workers01 -m command -a "mas-cli config generate --config /etc/mas/config.yaml" --become

# 7. Деплой workers и MAS
ansible-playbook playbooks/07_workers.yml

# 8. Element Web
ansible-playbook playbooks/08_element_web.yml

# 9. Разместить TLS сертификат на haproxy01
# scp combined.pem user@10.0.1.10:/etc/ssl/matrix/combined.pem
# scp fullchain.pem user@10.0.1.60:/etc/ssl/matrix/fullchain.pem
# scp privkey.pem user@10.0.1.60:/etc/ssl/matrix/privkey.pem

# 10. HAProxy
ansible-playbook playbooks/09_haproxy.yml

# 11. Верификация
ansible-playbook playbooks/verify.yml
```

## Создание первого администратора

```bash
# На synapse01
ssh synapse01
sudo -u matrix register_new_matrix_user \
  -c /etc/matrix-synapse/homeserver.yaml \
  -u admin \
  -p 'STRONG_PASSWORD' \
  -a   # -a = admin
```

## Обновление Synapse

```bash
# Обновить версию в inventory/host_vars/synapse01/main.yml и inventory/host_vars/workers01/main.yml
# synapse_version: "X.Y.Z"

ansible-playbook playbooks/06_synapse.yml
ansible-playbook playbooks/07_workers.yml
```

## Мониторинг

```bash
# HAProxy stats
open http://10.0.1.10:8404/stats

# Статус workers (на workers01)
systemctl list-units 'matrix-synapse-worker@*' --no-legend

# Логи воркера
journalctl -u matrix-synapse-worker@synchrotron1 -f

# Redis мониторинг
redis-cli -h 10.0.1.40 -a PASSWORD monitor

# PostgreSQL active connections
psql -h 10.0.1.30 -U postgres -c "SELECT count(*) FROM pg_stat_activity;"
```

## Откат воркера при проблеме

```bash
# Перезапустить конкретный воркер
ansible workers01 -m systemd -a "name=matrix-synapse-worker@synchrotron1 state=restarted" --become

# Перезапустить все воркеры
ansible workers01 -m systemd -a "name=matrix-synapse-workers.target state=restarted" --become
```

## Проверка federation

```bash
# С любого хоста
curl -s "https://matrix.org/_matrix/federation/v1/version" | jq .

# Проверить свой сервер
curl -s "https://matrix.example.com/_matrix/federation/v1/version" | jq .

# Тест federation с matrix.org
curl -s "https://federationtester.matrix.org/api/report?server_name=example.com" | jq .
```
