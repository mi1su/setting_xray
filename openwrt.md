# Настройка OpenWrt Snapshot 24\* для маршрутизации трафика через Xray с VLESS+Reality

Эта инструкция поможет вам настроить OpenWrt так, чтобы весь домашний трафик направлялся через удалённый сервер Xray, настроенный с использованием VLESS+Reality. OpenWrt будет выполнять роль клиента, маршрутизируя весь локальный трафик через сервер.

---

## Шаг 1: Установка и проверка Xray

### Убедитесь, что Xray установлен:

1. Выполним команду:

   ```bash
   xray -version
   ```

   Если Xray установлен, вы увидите информацию о версии.

2. Если Xray не установлен, установите его с помощью:
   ```bash
   apk update
   apk add xray-core
   ```
Если вдруг не хотите то можете 

---

## Шаг 2: Настройка Xray на OpenWrt как клиента

1. Создадим файл конфигурации:

   ```bash
   vi /etc/xray/config.json
   ```

2. Добавьте следующую конфигурацию:

   ```json
   {
     "log": {
       "loglevel": "warning"
     },
     "inbounds": [
       {
         "port": 1080,
         "listen": "127.0.0.1",
         "protocol": "socks",
         "settings": {
           "auth": "noauth",
           "udp": true
         },
         "tag": "socks-in"
       }
     ],
     "outbounds": [
       {
         "protocol": "vless",
         "settings": {
           "vnext": [
             {
               "address": "ВАШ_СЕРВЕРНЫЙ_IP_ИЛИ_ДОМЕН",
               "port": 443,
               "users": [
                 {
                   "id": "ВАШ-UUID",
                   "flow": "xtls-rprx-vision",
                   "encryption": "none"
                 }
               ]
             }
           ]
         },
         "streamSettings": {
           "network": "tcp",
           "security": "reality",
           "realitySettings": {
             "show": false,
             "dest": "www.google.com:443",
             "serverNames": ["www.google.com"],
             "privateKey": "ВАШ-ПРИВАТНЫЙ-КЛЮЧ-КЛИЕНТА",
             "shortIds": ["ВАШ-SHORTID"],
             "fingerprint": "chrome",
             "spiderX": "/"
           }
         },
         "tag": "vless-out"
       },
       {
         "protocol": "freedom",
         "settings": {},
         "tag": "direct"
       },
       {
         "protocol": "blackhole",
         "settings": {
           "response": {
             "type": "http"
           }
         },
         "tag": "block"
       }
     ],
     "routing": {
       "domainStrategy": "AsIs",
       "rules": [
         {
           "type": "field",
           "ip": ["geoip:private"],
           "outboundTag": "direct"
         },
         {
           "type": "field",
           "outboundTag": "vless-out",
           "port": "0-65535"
         }
       ]
     }
   }
   ```

3. Замените значения:

   - `ВАШ_СЕРВЕРНЫЙ_IP_ИЛИ_ДОМЕН`: IP-адрес или домен вашего сервера.
   - `ВАШ-UUID`: UUID клиента.
   - `ВАШ-ПРИВАТНЫЙ-КЛЮЧ-КЛИЕНТА`: Приватный ключ.
   - `ВАШ-SHORTID`: Short ID.
     www.google.com: ServerName, который вы указали на сервере (это должно совпадать).

   Инфо
   Порт: 1080 — это локальный порт, через который трафик будет передаваться в Xray. Локальный порт для SOCKS прокси (можете выбрать другой)
   listen: 127.0.0.1: Ограничивает доступ к SOCKS-прокси только локальным устройствам.
   udp: true: Включает поддержку UDP (важно для приложений, использующих DNS или стриминг).
   tag: "socks-in": Тег для идентификации входящего трафика (используется в routing).

"id": "ВАШ-UUID", // UUID, который вы настроили на сервере

    vless-out: Основной outbound, который направляет трафик на ваш сервер.
        address: IP-адрес или домен вашего сервера.
        port: Порт сервера (обычно 443 для VLESS+Reality).

         streamSettings: Настройки соединения (TCP, Reality).
        realitySettings: Параметры Reality (должны совпадать с сервером).

    direct: Outbound для прямого соединения (используется для локальных IP-адресов).
        Пример: Если вы хотите, чтобы локальные устройства (192.168.x.x) не шли через прокси, трафик направляется напрямую.

    blackhole: Используется для блокировки нежелательного трафика.

Параметр "fingerprint" в realitySettings используется для имитации реального HTTPS-клиента (например, браузера Chrome, Safari и т.д.) с определённым TLS-отпечатком. Это позволяет лучше маскировать трафик, чтобы он выглядел как реальный веб-трафик. В большинстве случаев для VLESS+Reality добавлять fingerprint не обязательно. Reality сама по себе хорошо справляется с обфускацией.

    domainStrategy: "AsIs":
        Указывает, что доменные имена не должны автоматически обрабатываться (используется прямой IP).

    Правила маршрутизации:
        Локальные IP-адреса (geoip:private) направляются через direct.
            Это позволяет устройствам в вашей локальной сети (192.168.x.x, 10.x.x.x) работать без прокси.
        Весь остальной трафик (port: 0-65535) направляется через vless-out.
            Это позволяет проксировать весь интернет-трафик через ваш удалённый сервер.

4. Включите автозапуск Xray:

   ```bash
   /etc/init.d/xray enable
   /etc/init.d/xray start
   ```

5. Проверьте работу Xray:

   ```bash
   netstat -tulpn | grep 1080
   ```

   ```bash
    tcp        0      0 :::1080                 :::*                    LISTEN      4405/xray
    udp        0      0 :::1080                 :::*                                4405/xray
   ```
Если нет 
1. Проверьте статус Xray:

   ```bash
   /etc/init.d/xray status
   ```
Если тут ошибка смотрим 

2. Просмотрите логи Xray:

   ```bash
   logread -e xray
   ```

   Если пусто можно руками запустить и в реальном времени смотреть 

  запустить Xray вручную из командной строки, чтобы увидеть, есть ли какие-либо ошибки, которые не отображаются при запуске через init.d.

   ```bash
/usr/bin/xray run -c /etc/xray/config.json
   ```

Еще просто помогает 
/etc/init.d/xray restart
---

## Шаг 3: Настройка перенаправления трафика

OpenWrt в SNAPSHOT-версии использует nftables вместо iptables для управления сетевыми правилами. Мы настроим перенаправление всего исходящего трафика через Xray.

1. Убедитесь, что nftables установлен:
   ```bash
   apk add nftables
   ```
4. Почему не стоит напрямую редактировать nftables?

   OpenWrt с fw4 автоматически генерирует и обновляет правила nftables из UCI.
   Любые изменения, сделанные вручную в nftables, могут быть перезаписаны при перезапуске fw4.
. Xray, работая на локальном порту 127.0.0.1:1080, не создает интерфейс (tun0) автоматически. Если вы используете Xray, перенаправление трафика должно быть настроено через прозрачный прокси (Transparent Proxy), а не через интерфейс.

Настройка без пользовательского скрипта

Если вы хотите сделать это полностью через UCI без скрипта, то можно настроить redir для перенаправления трафика:

    Создание зоны для Xray:
1. Настройка правил перенаправления:
```bash
# TCP
uci add firewall redirect
uci set firewall.@redirect[-1].name='Xray TCP Redirect'
uci set firewall.@redirect[-1].src='lan'
uci set firewall.@redirect[-1].proto='tcp'
uci set firewall.@redirect[-1].src_dport='0-65535'
uci set firewall.@redirect[-1].dest_port='1080'
uci set firewall.@redirect[-1].target='DNAT'

# UDP
uci add firewall redirect
uci set firewall.@redirect[-1].name='Xray UDP Redirect'
uci set firewall.@redirect[-1].src='lan'
uci set firewall.@redirect[-1].proto='udp'
uci set firewall.@redirect[-1].src_dport='0-65535'
uci set firewall.@redirect[-1].dest_port='1080'
uci set firewall.@redirect[-1].target='DNAT'

# Применение настроек
uci commit firewall
/etc/init.d/firewall restart
```

Этот способ проще и полностью интегрирован с fw4, что позволяет избежать конфликтов.

### Вариант 2: Через скрипт nftables

Создание правил перенаправления через UCI
Добавьте правила для исключения локального трафика и перенаправления остального через Xray.
Исключение локального трафика:

uci add firewall rule
uci set firewall.@rule[-1].name='BypassLocal'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].dest='lan'
uci set firewall.@rule[-1].proto='all'
uci set firewall.@rule[-1].target='ACCEPT'

Перенаправление всего TCP и UDP трафика через Xray:
1. Настройка интеграции с firewall:
```bash
uci add firewall include
uci set firewall.@include[-1].path='/etc/firewall.xray'
uci set firewall.@include[-1].type='script'
uci commit firewall
```
2. Настройка скрипта для nftables
Создайте скрипт `/etc/firewall.xray` для управления правилами nftables:
```bash
#!/bin/sh
nft add table inet xray

# Создаем цепочки для обработки трафика
nft add chain inet xray prerouting { type nat hook prerouting priority 0 \; }
nft add chain inet xray output { type nat hook output priority 0 \; }

# Исключаем локальную сеть (например, 192.168.1.0/24)
nft add rule inet xray prerouting ip saddr 192.168.1.0/24 accept
nft add rule inet xray prerouting ip daddr 192.168.1.0/24 accept
nft add rule inet xray output ip daddr 192.168.1.0/24 accept

# Исключаем localhost - трафик, предназначенный для самого роутера
nft add rule inet xray prerouting ip daddr 127.0.0.1 accept
nft add rule inet xray output ip daddr 127.0.0.1 accept

# Перенаправляем весь оставшийся TCP и UDP трафик через Xray
nft add rule inet xray prerouting meta l4proto tcp redirect to 1080
nft add rule inet xray prerouting meta l4proto udp redirect to 1080

Дайте права на исполнение:

```bash
chmod +x /etc/firewall.xray
```

## Шаг 3: Применение изменений

После настройки NAT и правил перезапустите службы:
Перезапустите firewall, чтобы применить новые правила:
```bash
uci commit
/etc/init.d/network restart
/etc/init.d/firewall restart
```
## Шаг 4: Проверка работы
Убедитесь, что правила работают:
Проверьте, что правила перенаправления трафика активны:
```bash
nft list ruleset
```

Или через UCI:
```bash
uci show firewall
```

3. Убедится, что внешний IP-адрес соответствует серверу, можно по [whatismyip.com](https://whatismyip.com) или [2ip](https://2ip.ru/)

4. Проверьте доступ к ранее недоступным ресурсам.
