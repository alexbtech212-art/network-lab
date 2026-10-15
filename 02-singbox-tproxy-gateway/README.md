# Transparent gateway on sing-box (TPROXY)

Прозрачный шлюз для гостевой сети (`br-guest`, `10.10.20.0/24`): весь трафик клиентов уходит в SOCKS5-прокси без настройки на устройствах.

**Платформа:** Raspberry Pi, ImmortalWrt 25.12, sing-box 1.12.25.

> Адреса, логины и пароли заменены заглушками.
> Итог проекта: этот вариант был заменён на mihomo из-за неработающего UDP через SOCKS5 ([проект 01](../01-mihomo-transparent-proxy/)). Документ сохранён, потому что содержит ценный разбор TPROXY и найденных проблем.

## Выбор архитектуры: TUN или TPROXY

**TUN.** sing-box поднимает виртуальный интерфейс. Но TUN inbound с `auto_route: false` не обрабатывает транзитный трафик (не от самого роутера) — это открытый баг sing-box (issue #3274). Через `tcpdump` было видно: пакеты клиента доходили до `tun0`, но sing-box их игнорировал и генерировал паразитные внутренние SYN с мгновенными RST.

**TPROXY.** Штатный механизм ядра Linux для прозрачного перехвата транзитного трафика. Выбран как основной.

Как это работает: ядро в prerouting ещё до решения о маршрутизации перехватывает пакет и помечает его fwmark; policy routing направляет его в таблицу 100 с маршрутом `type=local`; ядро доставляет пакет локальному listener на `127.0.0.1:2080`, сохраняя оригинальный destination (sing-box читает его через `IP_ORIGDSTADDR`). Маршрут должен быть именно `local`, а не unicast: unicast заставил бы ядро форвардить пакет, а не доставлять локально.

## Архитектура

```
Клиент (br-guest)
  → nftables TPROXY (mangle prerouting, priority -1)
    → sing-box 127.0.0.1:2080
      → SOCKS5-прокси
        → интернет

DNS:
Клиент → dnsmasq (10.10.20.1:53)
  → dnscrypt-proxy (127.0.0.1:5353)
    → DoH через SOCKS5
      → Cloudflare 1.1.1.1
```

## Ключевые конфиги

### sing-box (`/etc/sing-box/config.json`)

```json
{
  "log": { "level": "info", "timestamp": true },
  "dns": {
    "servers": [{
      "tag": "proxy-dns",
      "type": "https",
      "server": "1.1.1.1",
      "detour": "proxy-out"
    }],
    "rules": [{ "outbound": "any", "server": "proxy-dns" }],
    "final": "proxy-dns",
    "strategy": "ipv4_only"
  },
  "inbounds": [
    { "type": "tproxy", "tag": "tproxy-in",     "listen": "127.0.0.1", "listen_port": 2080, "network": "tcp", "sniff": true },
    { "type": "tproxy", "tag": "tproxy-in-udp", "listen": "127.0.0.1", "listen_port": 2080, "network": "udp", "sniff": true }
  ],
  "outbounds": [
    { "type": "socks", "tag": "proxy-out", "server": "HOST", "server_port": 0, "username": "USER", "password": "PASS", "version": "5" },
    { "type": "direct", "tag": "direct-out" },
    { "type": "block",  "tag": "block-out" }
  ],
  "route": {
    "rules": [{ "ip_is_private": true, "outbound": "direct-out" }],
    "auto_detect_interface": false,
    "default_domain_resolver": "proxy-dns",
    "final": "proxy-out"
  }
}
```

### nftables (`/etc/nftables.d/proxy-tproxy.nft`)

```nft
chain guest_tproxy_prerouting {
    type filter hook prerouting priority mangle - 1; policy accept;
    iifname != "br-guest" return
    fib daddr type { local, broadcast, multicast } return
    udp dport { 67, 68 } return
    ip daddr 10.10.20.1 return
    meta l4proto tcp counter tproxy ip to 127.0.0.1:2080 meta mark set 0x1
    meta l4proto udp counter tproxy ip to 127.0.0.1:2080 meta mark set 0x1
}
```

### Policy routing (UCI)

```sh
uci add network route
uci set network.@route[-1].interface='loopback'
uci set network.@route[-1].target='0.0.0.0/0'
uci set network.@route[-1].table='100'
uci set network.@route[-1].type='local'
uci commit network
```

Проверка: `ip route show table 100` → `local default dev lo proto static scope host`.

### dnsmasq (`/etc/dnsmasq.conf`, добавлено)

```
interface=br-guest
bind-interfaces
```

### dnscrypt-proxy2 (`dnscrypt-proxy.toml`, основное)

```toml
listen_addresses = ['127.0.0.1:5353']
proxy = 'socks5://USER:PASS@HOST:PORT'
server_names = ['cloudflare', 'cloudflare-ipv4']
ipv6_servers = false
dnscrypt_servers = false
doh_servers = true
```

### Пакеты

```sh
apk add kmod-nft-tproxy   # поддержка TPROXY в ядре
apk add jq                # правка JSON-конфигов
apk add dnscrypt-proxy2   # DoH через SOCKS5
```

### Скрипт смены прокси

`proxy-switch HOST PORT USER PASS` — правит `config.json` через `jq`, проверяет `sing-box check`, перезапускает службу.

## Решённые проблемы

| # | Проблема | Причина → решение |
|---|---|---|
| 1 | `nft --check` падает с `No such file or directory` | Не установлен модуль ядра → `apk add kmod-nft-tproxy` |
| 2 | Ошибка `specify 'tproxy ip' or 'tproxy ip6'` | В таблице `inet` нужно явно писать `tproxy ip to` |
| 3 | `redefinition of symbol` | fw4 дублирует обработку include-файлов; `chain` дублируется безвредно, `define` — нет. Убрал `define`, список reserved IP заменил на `fib daddr type { local, broadcast, multicast }` |
| 4 | DHCP перехватывался TPROXY (`socks5: request rejected, code=1`) | Добавил `udp dport { 67, 68 } return` |
| 5 | Запросы к `10.10.20.1:53` уходили в sing-box вместо dnsmasq | Добавил `ip daddr 10.10.20.1 return` до правил TPROXY |
| 6 | Старые файлы от TUN-конфигурации ломали DNS | `proxy-mark.nft` и `proxy-killswitch.nft` ставили fwmark на весь трафик `br-guest` → удалил |
| 7 | `FATAL: unknown field "client_subnet"` | Поле появилось в более новых версиях sing-box → убрал |
| 8 | dnsmasq не отвечал на `br-guest` | Сочетание `localservice='1'`, `nonwildcard='1'` и отсутствия `interface=br-guest` → добавил `interface` и `bind-interfaces`, выставил `localservice='0'`, `nonwildcard='0'` |
| 9 | `br-guest` попадал в зону `lan` | UCI-имя сети `guestnet`, а зона была привязана к `guest` → `uci set firewall.@zone[3].network='guestnet'` |

### Kill switch работает нативно

При остановленном sing-box пакет получает mark `0x1`, идёт в таблицу 100, ядро пытается доставить его локально на `127.0.0.1:2080`, где никто не слушает → RST. В TUN-архитектуре был риск fallback на основную таблицу маршрутизации. Проверено: `kill $(pgrep sing-box)` → на клиенте `curl: (35) Recv failure: Connection reset by peer`.

## Нерешённое

- **Нестабильность dnsmasq после `fw4 reload`.** При перезапусках иногда dnsmasq переставал отвечать на `br-guest`, пока не применялся `bind-interfaces`. Возможен конфликт между UCI-настройками и ручными правками `/etc/dnsmasq.conf`; причина не исследована.
- **ECS (client_subnet).** Нужен, чтобы ответы DNS учитывали геолокацию прокси. В sing-box 1.12.25 не поддерживается; обновление требует ручной сборки.
- **Захардкоженные DNS (8.8.8.8).** Некоторые приложения обращаются к `8.8.8.8:53` напрямую. Решение: перехват UDP/TCP 53 правилом NAT-redirect на локальный DNS (реализовано позже в проекте 01).
- **Откат бэкапа.** Восстановление через веб-интерфейс возвращает старые nft-файлы и привязки зон. Вывод: бэкап делать после всех правок.

## Полезные команды

```sh
# Применить nft-файлы
fw4 stop; nft delete table inet fw4 2>/dev/null; fw4 start

# Кто слушает порт 53
netstat -ulnp | grep ":53 "

# Проверка с клиента
curl -v https://1.1.1.1 --max-time 10
nslookup google.com
```
