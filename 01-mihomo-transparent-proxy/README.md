# Transparent gateway on mihomo (Raspberry Pi + Windows)

Прозрачный шлюз: трафик устройств идёт через SOCKS5-прокси без какого-либо клиента на самих устройствах. Сначала на роутере Raspberry Pi 5 (гостевая сеть), затем на ПК с Windows.

> Все адреса, логины и пароли заменены заглушками (`PROXY_IP`, `LOGIN`, `PASSWORD`).

## Окружение

- Raspberry Pi 5, ImmortalWrt 25.12 (aarch64_cortex-a76), пакетный менеджер `apk`
- Гостевая сеть `br-guest`, `10.10.20.0/24`; роутер стоит за другим роутером (двойной NAT)
- SOCKS5-прокси с поддержкой UDP

## Схема

```
Клиент гостевой сети
  → nftables (TPROXY) перехватывает TCP и UDP
    → mihomo (порт 7893)
      → SOCKS5-прокси
        → интернет

DNS (порт 53): правило NAT перенаправляет на DNS mihomo (порт 1053),
который резолвит имена через тот же прокси.
```

## Почему mihomo, а не sing-box

Сначала использовался sing-box ([см. проект 02](../02-singbox-tproxy-gateway/)). В нём не заработала передача UDP через SOCKS5 (UDP ASSOCIATE), из-за чего не работал QUIC. mihomo передаёт UDP через SOCKS5 корректно, поэтому выбран он.

## Конфигурация mihomo (Raspberry Pi)

```yaml
tproxy-port: 7893
mixed-port: 7890

dns:
  enable: true
  listen: 0.0.0.0:1053
  ipv6: false
  enhanced-mode: redir-host
  default-nameserver:
    - 1.1.1.1
  nameserver:
    - https://8.8.8.8/dns-query#PROXY

proxies:
  - name: PROXY
    type: socks5
    server: PROXY_IP
    port: PROXY_PORT
    username: LOGIN
    password: PASSWORD
    udp: true

proxy-groups:
  - name: MAIN
    type: select
    proxies: [PROXY, DIRECT]

rules:
  - IP-CIDR,192.168.0.0/16,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT
  - IP-CIDR,172.16.0.0/12,DIRECT
  - IP-CIDR,127.0.0.0/8,DIRECT
  - MATCH,MAIN
```

- `tproxy-port` должен совпадать с портом в правиле nftables.
- `redir-host` вместо `fake-ip`: клиенту отдаются настоящие IP, это важно для QUIC.
- `#PROXY` у DNS-сервера: сами DNS-запросы тоже идут через прокси.
- `udp: true` разрешает UDP через прокси (QUIC).

## Правила nftables

Файл `/etc/nftables.d/proxy-tproxy.nft`:

```nft
chain guest_tproxy_prerouting {
    type filter hook prerouting priority mangle - 1; policy accept;
    ip saddr != 10.10.20.0/24 return
    fib daddr type { local, broadcast, multicast } return
    udp dport { 67, 68 } return
    ip daddr 10.10.20.1 return
    udp dport 53 return
    tcp dport 53 return
    meta l4proto tcp tproxy ip to 127.0.0.1:7893 meta mark set 1
    meta l4proto udp tproxy ip to 127.0.0.1:7893 meta mark set 1
}

chain guest_dns_dstnat {
    type nat hook prerouting priority dstnat; policy accept;
    ip saddr != 10.10.20.0/24 return
    udp dport 53 redirect to :1053
    tcp dport 53 redirect to :1053
}
```

Первая цепочка перехватывает только трафик гостевой подсети и пропускает локальные и широковещательные адреса, DHCP, обращения к самому роутеру и DNS. Вторая перенаправляет DNS гостей на mihomo. Для TPROXY нужен ещё policy routing: пакеты с меткой 1 доставляются локально. Применение: `fw4 restart`.

## Проблемы и решения

### 1. У клиентов нет интернета, хотя mihomo «running»
**Причина:** `tproxy-port` и `mixed-port` оба были заданы одним значением (порт прокси). В логе: `bind: address already in use`, плюс mihomo слушал не тот порт, куда nftables отправлял трафик.
**Решение:** привести порты в соответствие с nftables (7893 и 7890) и перезапустить службу. В логе должно быть `TProxy server listening at: 127.0.0.1:7893` без ошибок bind.

### 2. Сам роутер не резолвит имена, `apk update` падает с «Operation not permitted»
**Причина:** у dnsmasq был `no-resolv` и единственный апстрим `127.0.0.1#5353`, где никто не слушал. Ошибка wget вводила в заблуждение: на деле не разрешалось имя.
**Диагностика:** `nslookup имя 1.1.1.1` работает, `nslookup имя 127.0.0.1` нет.
**Решение:**
```sh
uci add_list dhcp.@dnsmasq[0].server='1.1.1.1'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

### 3. Таймауты до прокси-сервера
В логе: `dial tcp ...: connect: connection timed out`. nftables и mihomo при этом исправны.
**Порядок проверки:** сначала напрямую с роутера, в обход mihomo:
```sh
curl -v -x socks5h://LOGIN:PASSWORD@PROXY_IP:PROXY_PORT https://example.com
```
Пришла страница — прокси жив, ищем проблему в конфиге. Таймаут — виноват прокси или путь до него.
**Вывод:** недоступный прокси выглядит так же, как сломанный TPROXY, поэтому прокси проверяется первым. Перед заменой прокси делается бэкап конфига. Причина временной недоступности не установлена.

### 4. DNS-тест показывает два резолвера
Один — из подсети самого прокси-провайдера, второй — Google 8.8.8.8 из `nameserver`. Домашний провайдер в списке не появлялся, то есть утечки нет.

### Особенности окружения роутера
- Нет `ss` — используется `netstat -tlnp`.
- `nc` из BusyBox не понимает `-z` и `-w`; для проверок удобнее `curl -x socks5h://...`.
- В `ash` запись `<client-ip>` — это перенаправление ввода, подставлять нужно реальный IP без скобок.
- Связка `fw4 stop && nft delete table inet fw4 && fw4 start` здесь не работает, использовать `fw4 restart`.

## Windows: mihomo в режиме TUN

На Windows нет TPROXY (он только в Linux), поэтому трафик перехватывает виртуальный адаптер TUN (бинарник mihomo + `wintun.dll`, запуск от администратора).

```yaml
mixed-port: 7890
allow-lan: false
log-level: info
tcp-concurrent: true
unified-delay: true

tun:
  enable: true
  stack: system
  auto-route: true
  auto-detect-interface: true
  strict-route: false
  dns-hijack:
    - any:53
  mtu: 1500

dns:
  enable: true
  enhanced-mode: redir-host
  nameserver:
    - tcp://8.8.8.8#PROXY

sniffer:
  enable: true
  sniff:
    TLS:
      ports: [443]
    HTTP:
      ports: [80]
  override-destination: true
  skip-domain:
    - "+.local"
    - "+.msftconnecttest.com"

proxies:
  - name: MAIN
    type: socks5
    server: PROXY_IP
    port: PROXY_PORT
    username: LOGIN
    password: PASSWORD
    udp: true

proxy-groups:
  - name: PROXY
    type: select
    proxies: [MAIN]

rules:
  - MATCH,PROXY
```

### Настройки Windows вне конфига
- Отключить IPv6 на адаптере: `Disable-NetAdapterBinding -Name Wi-Fi -ComponentID ms_tcpip6`
- Вручную прописать на адаптере внешний DNS (например, 1.1.1.1).
- Отключить DoH/DoT/DDR: `netsh dnsclient set global doh=no dot=no ddr=no`
- Отключить SMHNR: `DisableSmartNameResolution = 1` в `HKLM\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient`, затем перезагрузка.
- Если установлен другой VPN-клиент, полностью выйти из него.

### Проблемы Windows
- **DNS идёт мимо туннеля.** Windows направляет трафик локальной подсети и loopback в обход TUN, `dns-hijack` его не видит. Решение: прописать на адаптере внешний IP; внутри TUN `any:53` перехватит запрос, а реальный резолвер определяется строкой `nameserver`.
- **fake-ip ломает QUIC.** Первый UDP-пакет уходит на fake-ip раньше, чем sniffer подставит настоящий адрес; записи вытесняются из кеша ограниченного размера. Решение: `redir-host`.
- **`strict-route: true` ломает связь.** Воспроизводилось повторно даже после очистки окружения (вероятно, остатки фильтров WFP от другого VPN-клиента). Оставлено `false`, а защиту от утечек обеспечивают отключённый IPv6 и внешний DNS на адаптере.
- **Выбор DNS.** Адрес резолвера из подсети прокси давал непредсказуемую геолокацию, хотя трафик шёл верно (проверено логом). Замена на `tcp://8.8.8.8#PROXY` дала стабильный результат.
- **Зомби-процессы.** После некорректных перезапусков висящие процессы mihomo давали симптом «всё отправляется, ничего не возвращается». Перед диагностикой проверять, что запущен ровно один процесс. При ошибке `Cannot create a file when that file already exists` помогает перезагрузка.

## Дальнейшее развитие

Для защиты от наблюдения на стороне домашней сети арендован VDS: ПК → VDS (Shadowsocks) → SOCKS5-прокси → интернет. Цепочка через `dialer-proxy` на ПК ломала DNS и QUIC, поэтому второй mihomo вынесен на VDS. Добавлено правило брандмауэра, блокирующее исходящий трафик Wi-Fi адаптера ко всем публичным IP, кроме адреса VDS: при остановке mihomo интернет пропадает, а не уходит напрямую.

## Выводы

- Почти все случаи «нет интернета» сводились к трём причинам: рассинхрон портов между nftables и mihomo, сломанный DNS на самом роутере, недоступный прокси.
- Порядок диагностики: логи запуска (порты) → проверка прокси напрямую через `curl -x socks5h` → DNS.
- При смене прокси меняется только блок `proxies:`; перед правкой делается бэкап.
