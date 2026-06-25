# Operations Guide — Matrix Stack (Docker)

## Сервисы и где они запущены

| Хост        | Сервис              | Тип           | Compose dir                      |
|-------------|---------------------|---------------|----------------------------------|
| haproxy01   | HAProxy             | native        | —                                |
| synapse01   | Synapse main        | Docker        | /opt/matrix/compose/synapse/     |
| postgres01  | PostgreSQL 16       | Docker        | /opt/matrix/compose/postgres/    |
| redis01     | Redis 7.2           | Docker        | /opt/matrix/compose/redis/       |
| livekit01   | LiveKit + lk-jwt    | Docker        | /opt/matrix/compose/livekit/     |
| coturn01    | Coturn              | Docker        | /opt/matrix/compose/coturn/      |
| workers01   | 17 Synapse workers  | Docker        | /opt/matrix/compose/workers/     |
| workers01   | MAS                 | Docker        | /opt/matrix/compose/mas/         |
| element01   | Element Web (nginx) | Docker        | /opt/matrix/compose/element/     |

## Первый деплой

```bash
# 0. Зависимости Ansible
ansible-galaxy collection install -r requirements.yml

# 1. Заполнить vault-секреты
cp inventory/group_vars/all/vault.yml vault_template.yml
# Отредактировать все CHANGE_ME значения
ansible-vault encrypt inventory/group_vars/all/vault.yml
echo "your_vault_password" > .vault_pass && chmod 600 .vault_pass

# 2. Установить реальные IP в inventory/group_vars/all/main.yml
#    external_ip, haproxy_ip, synapse_ip, ...

# 3. Проверить коннективность
ansible all -m ping

# 4. Пошаговый деплой (рекомендуется при первом запуске)
ansible-playbook playbooks/01_common.yml
ansible-playbook playbooks/02_docker.yml
ansible-playbook playbooks/03_postgres.yml
ansible-playbook playbooks/04_redis.yml
ansible-playbook playbooks/05_coturn.yml
ansible-playbook playbooks/06_livekit.yml
ansible-playbook playbooks/07_synapse.yml  # генерирует signing key

# 5. Скопировать signing key с synapse01 на workers01
ansible synapse01 -m fetch -a "src={{ synapse_config_dir }}/{{ matrix_server_name }}.signing.key dest=/tmp/"
ansible workers01 -m copy -a "src=/tmp/synapse01{{ synapse_config_dir }}/{{ matrix_server_name }}.signing.key dest={{ synapse_config_dir }}/{{ matrix_server_name }}.signing.key owner={{ matrix_uid }} mode=0640"

# 6. Workers и MAS
ansible-playbook playbooks/08_workers.yml

# 7. Element Web
ansible-playbook playbooks/09_element_web.yml

# 8. TLS сертификаты
# На haproxy01:
cat fullchain.pem privkey.pem > /etc/ssl/matrix/combined.pem
chmod 640 /etc/ssl/matrix/combined.pem && chown root:haproxy /etc/ssl/matrix/combined.pem

# На coturn01:
cp fullchain.pem /etc/ssl/matrix/fullchain.pem
cp privkey.pem   /etc/ssl/matrix/privkey.pem

# 9. HAProxy (только когда TLS cert готов)
ansible-playbook playbooks/10_haproxy.yml

# 10. Полная верификация
ansible-playbook playbooks/verify.yml
```

## Управление контейнерами

```bash
# Просмотр всех Matrix контейнеров на хосте
docker ps --filter "name=matrix-"
docker ps --filter "name=synapse-"

# Логи конкретного воркера
docker logs synapse-synchrotron1 -f --tail=100

# Логи MAS
docker logs matrix-mas -f

# Рестарт конкретного воркера
docker restart synapse-synchrotron1

# Рестарт всех воркеров
docker compose -f /opt/matrix/compose/workers/docker-compose.yml restart

# Рестарт Synapse main
docker compose -f /opt/matrix/compose/synapse/docker-compose.yml restart

# Статус всех воркеров
docker compose -f /opt/matrix/compose/workers/docker-compose.yml ps
```

## Обновление Synapse

```bash
# 1. Обновить synapse_version в inventory/host_vars/synapse01/main.yml
#    и в inventory/host_vars/workers01/main.yml

# 2. Применить на main процессе
ansible-playbook playbooks/07_synapse.yml

# 3. Применить на воркерах
ansible-playbook playbooks/08_workers.yml

# Или вручную:
docker compose -f /opt/matrix/compose/synapse/docker-compose.yml pull
docker compose -f /opt/matrix/compose/synapse/docker-compose.yml up -d

docker compose -f /opt/matrix/compose/workers/docker-compose.yml pull
docker compose -f /opt/matrix/compose/workers/docker-compose.yml up -d
```

## Создание первого администратора

```bash
# На synapse01
docker exec -it matrix-synapse register_new_matrix_user \
  -c /etc/matrix-synapse/homeserver.yaml \
  -u admin \
  -p 'STRONG_PASSWORD' \
  -a \
  http://localhost:8008
```

## Просмотр логов PostgreSQL

```bash
# На postgres01
docker logs matrix-postgres -f --tail=200

# Или из примонтированной директории
tail -f /var/log/matrix/postgres/postgresql.log
```

## Мониторинг Redis

```bash
# На redis01
docker exec -it matrix-redis redis-cli -h 10.0.1.40 -a PASSWORD
> INFO stats
> INFO memory
> CLIENT LIST
```

## Проверка federation

```bash
curl -s "https://matrix.example.com/_matrix/federation/v1/version" | jq .
# Внешний тестер:
curl "https://federationtester.matrix.org/api/report?server_name=example.com" | jq .
```

## Откат на предыдущую версию

```bash
# Docker сохраняет предыдущий образ под тегом <none>
# Откат:
docker tag ghcr.io/element-hq/synapse:v1.121.0 ghcr.io/element-hq/synapse:current
docker tag ghcr.io/element-hq/synapse:v1.120.0 ghcr.io/element-hq/synapse:v1.121.0
docker compose -f /opt/matrix/compose/synapse/docker-compose.yml up -d
```
