# Network Layout — Matrix Stack (behind NAT)

## Топология

```
Internet
    │
    │  TCP 80, 443, 8448
    ▼
┌─────────────────────────────────────────────┐
│              NAT Gateway / Router            │
│   External IP: 203.0.113.10                 │
│                                              │
│  DNAT rules:                                 │
│    80   → 10.0.1.10:80                      │
│    443  → 10.0.1.10:443                     │
│    8448 → 10.0.1.10:8448                    │
│    3478 → 10.0.1.60:3478 (TURN UDP+TCP)     │
│    5349 → 10.0.1.60:5349 (TURNS UDP+TCP)    │
│    49152-65535/udp → 10.0.1.60 (TURN relay) │
│    7881 → 10.0.1.50:7881 (LiveKit TCP)      │
│    7882/udp → 10.0.1.50:7882 (LiveKit UDP)  │
└─────────────────────────────────────────────┘
                    │
                    │  Internal: 10.0.1.0/24
                    │
    ┌───────────────┼───────────────────────┐
    │               │                       │
    ▼               ▼                       ▼
┌──────────┐   ┌──────────┐           ┌──────────┐
│ haproxy01│   │coturn01  │           │livekit01 │
│10.0.1.10 │   │10.0.1.60 │           │10.0.1.50 │
└──────────┘   └──────────┘           └──────────┘
    │  :8008  :8448 federation
    │  routes ──────────────────────────────────────────┐
    │                                                    │
    ├─── /sync ────────────────────────────────────┐    │
    │                                              ▼    ▼
    │                                         ┌──────────┐
    │                                         │workers01 │
    │                                         │10.0.1.70 │
    │                                         │          │
    │                                         │synchrotron│
    │                                         │fed_reader │
    │                                         │generic   │
    │                                         │media     │
    │                                         │MAS :8090 │
    │                                         └──────────┘
    │                                              │
    │                    replication (9093)        │
    │                    ◄────────────────────────┘
    │
    ├─── admin ──────────────────────────────► synapse01
    │                                          10.0.1.20
    │                                              │
    │                                              │ psql
    │                                              ▼
    │                                         ┌──────────┐
    │                                         │postgres01│
    │                                         │10.0.1.30 │
    │                                         └──────────┘
    │                                              ▲
    │                                    (MAS also │)
    │                                         workers01
    │
    └─── element ────────────────────────────► element01
                                               10.0.1.80
```

## Таблица портов

| Хост        | IP          | Порт       | Протокол | Источник          | Назначение                    |
|-------------|-------------|------------|----------|-------------------|-------------------------------|
| haproxy01   | 10.0.1.10   | 80         | TCP      | Internet          | HTTP → HTTPS redirect         |
| haproxy01   | 10.0.1.10   | 443        | TCP      | Internet          | HTTPS Matrix + Element + MAS  |
| haproxy01   | 10.0.1.10   | 8448       | TCP      | Internet          | Matrix Federation             |
| haproxy01   | 10.0.1.10   | 8404       | TCP      | Internal only     | HAProxy stats                 |
| synapse01   | 10.0.1.20   | 8008       | TCP      | haproxy01         | Synapse HTTP API              |
| synapse01   | 10.0.1.20   | 9093       | TCP      | workers01         | Synapse replication           |
| synapse01   | 10.0.1.20   | 9101       | TCP      | Internal only     | Prometheus metrics            |
| postgres01  | 10.0.1.30   | 5432       | TCP      | synapse01,workers | PostgreSQL                    |
| redis01     | 10.0.1.40   | 6379       | TCP      | synapse01,workers | Redis                         |
| livekit01   | 10.0.1.50   | 7880       | TCP      | haproxy01         | LiveKit WebSocket             |
| livekit01   | 10.0.1.50   | 7881       | TCP      | Internet          | LiveKit WebRTC TCP            |
| livekit01   | 10.0.1.50   | 7882       | UDP      | Internet          | LiveKit WebRTC UDP            |
| livekit01   | 10.0.1.50   | 8181       | TCP      | haproxy01         | LiveKit JWT service           |
| coturn01    | 10.0.1.60   | 3478       | TCP+UDP  | Internet          | STUN/TURN                     |
| coturn01    | 10.0.1.60   | 5349       | TCP+UDP  | Internet          | STUNS/TURNS (TLS)             |
| coturn01    | 10.0.1.60   | 49152-65535| UDP      | Internet          | TURN relay ports              |
| workers01   | 10.0.1.70   | 8101-8104  | TCP      | haproxy01         | Synchrotron workers           |
| workers01   | 10.0.1.70   | 8111-8112  | TCP      | haproxy01         | Federation reader workers     |
| workers01   | 10.0.1.70   | 8121-8124  | TCP      | haproxy01         | Generic workers               |
| workers01   | 10.0.1.70   | 8131       | TCP      | haproxy01         | Media worker                  |
| workers01   | 10.0.1.70   | 8141-8142  | TCP      | synapse01         | Event persisters (replication)|
| workers01   | 10.0.1.70   | 8151-8155  | TCP      | synapse01         | Stream writers (replication)  |
| workers01   | 10.0.1.70   | 8161       | TCP      | synapse01         | Background tasks              |
| workers01   | 10.0.1.70   | 8171       | TCP      | synapse01         | Pusher                        |
| workers01   | 10.0.1.70   | 8181-8182  | TCP      | synapse01         | Federation senders            |
| workers01   | 10.0.1.70   | 8090       | TCP      | haproxy01         | MAS                           |
| element01   | 10.0.1.80   | 80         | TCP      | haproxy01         | Element Web (nginx)           |

## NAT правила (iptables / nftables)

```bash
# Пример для iptables PREROUTING на шлюзе
iptables -t nat -A PREROUTING -p tcp --dport 80   -j DNAT --to-destination 10.0.1.10:80
iptables -t nat -A PREROUTING -p tcp --dport 443  -j DNAT --to-destination 10.0.1.10:443
iptables -t nat -A PREROUTING -p tcp --dport 8448 -j DNAT --to-destination 10.0.1.10:8448

# TURN/STUN
iptables -t nat -A PREROUTING -p tcp --dport 3478 -j DNAT --to-destination 10.0.1.60:3478
iptables -t nat -A PREROUTING -p udp --dport 3478 -j DNAT --to-destination 10.0.1.60:3478
iptables -t nat -A PREROUTING -p tcp --dport 5349 -j DNAT --to-destination 10.0.1.60:5349
iptables -t nat -A PREROUTING -p udp --dport 5349 -j DNAT --to-destination 10.0.1.60:5349
iptables -t nat -A PREROUTING -p udp --dport 49152:65535 -j DNAT --to-destination 10.0.1.60

# LiveKit WebRTC
iptables -t nat -A PREROUTING -p tcp --dport 7881 -j DNAT --to-destination 10.0.1.50:7881
iptables -t nat -A PREROUTING -p udp --dport 7882 -j DNAT --to-destination 10.0.1.50:7882

# MASQUERADE для внутренней сети
iptables -t nat -A POSTROUTING -s 10.0.1.0/24 -j MASQUERADE
```

## DNS записи

```
; Основные
matrix.example.com.     A    203.0.113.10
auth.example.com.       A    203.0.113.10
element.example.com.    A    203.0.113.10
livekit.example.com.    A    203.0.113.10

; Federation well-known через SRV или через matrix.example.com
; (если Synapse слушает 8448 через HAProxy, SRV не нужен)
; Используем well-known delegation:
; GET https://example.com/.well-known/matrix/server → {"m.server":"matrix.example.com:443"}

; TURN/STUN — тот же внешний IP
turn.example.com.       A    203.0.113.10
```

## .well-known файлы

Synapse раздаёт их сам (serve_server_wellknown: true):

- `https://example.com/.well-known/matrix/server` → `{"m.server":"matrix.example.com:443"}`
- `https://example.com/.well-known/matrix/client` → `{"m.homeserver":{"base_url":"https://matrix.example.com"}}`

HAProxy маршрутизирует `/.well-known/matrix/` на Synapse main.

## TLS Сертификат

Все домены должны быть покрыты единым wildcard или SAN сертификатом:
- `*.example.com` (wildcard)

Или отдельные сертификаты для каждого домена (Let's Encrypt).

Формат для HAProxy — объединённый PEM:
```bash
cat fullchain.pem privkey.pem > /etc/ssl/matrix/combined.pem
chmod 640 /etc/ssl/matrix/combined.pem
chown root:haproxy /etc/ssl/matrix/combined.pem
```

Для Coturn нужны отдельные файлы:
```bash
cp fullchain.pem /etc/ssl/matrix/fullchain.pem
cp privkey.pem   /etc/ssl/matrix/privkey.pem
```
