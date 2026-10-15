# Network & Linux Lab

Домашняя лаборатория по сетям и Linux: прозрачные шлюзы на Raspberry Pi (ImmortalWrt) и Windows, маршрутизация трафика, DNS, файрвол, диагностика. Все проекты сделаны самостоятельно и задокументированы вместе с ошибками и их разбором.

**Languages:** русский (родной), română (fluent), English (technical reading/writing).

## Проекты

| Проект | Что внутри | Стек |
|---|---|---|
| [01 — Transparent gateway on mihomo](01-mihomo-transparent-proxy/) | Прозрачный шлюз для гостевой сети на Raspberry Pi и клиент в режиме TUN на Windows, защита от DNS-утечек, kill switch | ImmortalWrt, nftables, TPROXY, mihomo, PowerShell |
| [02 — Transparent gateway on sing-box (TPROXY)](02-singbox-tproxy-gateway/) | Перехват транзитного трафика на уровне ядра, policy routing, DoH-резолвер, разбор 10+ проблем | ImmortalWrt, nftables, sing-box, dnsmasq, dnscrypt-proxy |

## Чему эти проекты меня научили

- Диагностика «нет интернета» по слоям: логи → прямая проверка соединения → DNS.
- Как работают TPROXY, fwmark и policy routing (маршрут `type=local`), и почему TUN не подходит для транзитного трафика.
- DNS: перехват, DoH, утечки, различия `fake-ip` и `redir-host`.
- nftables: цепочки, хуки, приоритеты, взаимодействие с fw4 в OpenWrt.
- Kill switch: как сделать так, чтобы при падении службы трафик не уходил напрямую.
- Ведение документации: что сделано, почему, что не решено.

## English summary

Two hands-on projects building transparent traffic gateways. Project 1 uses mihomo with nftables TPROXY on a Raspberry Pi 5 (ImmortalWrt) and TUN mode on Windows. Project 2 uses sing-box with kernel-level TPROXY, policy routing and a DoH resolver. Each write-up covers architecture, configs (secrets redacted), troubleshooting steps and unresolved issues.

## Контакты

- Email: `your@email`
- LinkedIn: `ссылка`
- Локация: Кишинёв, Молдова
