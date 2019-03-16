### Как настроить ipv6 tunnel в Mikrotik (RouterOS)

#### 1. Начальные условия

Провайдер предоставляет Internet через L2TP соединение.
ip адрес прямой без NAT, динамический - меняется раз в сутки (или при переподключении L2TP соединения)

#### 2. Значение MTU и IPv6 Fragmentation

Значение MTU означает максимальный размер фрейма (PDU) который может быть передан между устройствами. Стандартное значение MTU для Ethernet `1500` байт. Когда подключение к интернету происходит не напрямую, а например через L2TP как у меня, часть MTU (`40` байт) расходуется на заголовки для L2TP туннеля. Поэтому значение MTU для соединения с  Internet снижается до `1460` (`1500 - 40`).

Для создания ipv6 туннеля (Protocol 41) используется ещё `20` байт, поэтому ipv6 соединения будут использовать MTU = `1440` (`1500 - 40 - 20`).

Если размер пакета не помещается в MTU, тогда такой пакет должен быть фрагментирован. Промежуточные узлы работающие по IPv4 протоколу могут (но не обязаны) самостоятельно фрагментировать пакеты.

IPv6 не поддерживает фрагментацию на промежуточных узлах, и если промежуточное устройство получит пакет который превышает MTU, то оно отбросит пакет и вернёт ICMP сообщение `Type 2 - Packet Too Big`.

Поэтому для нормальной работы IPv6 важно чтобы ICMP сообщения не блокировались на Firewall. А если блокируются, то следуйте рекомендациям https://tools.ietf.org/html/rfc4890

Так же желательно знать ваше значение MTU для Internet соединения.

Можно не возиться с определением MTU для ipv6 соединений и использовать `1280` - минимальное возможное значение MTU для ipv6.

#### 3. Определение MTU для соединения с Internet

Или считаем значение MTU используя известную информацию о протоколах например для L2TP

`1500 - 40 = 1460`

Или подбираем значение используя команду (Linux)

`ping -M do -s 1500 1.1.1.1`.

Подбираем максимальный размер пакета при котором команда `ping` выполняется успешно.
Команда `ping` может вернуть ошибку как эта

`ping: local error: Message too long, mtu=1460`.

В которой сразу указано доступное значение MTU.

Важно помнить что для Linux `-s 1500` это размер пакета без заголовков (ip + icmp `28` байт). Реальный размер пакета будет равен (`1500 + 28`).

#### 4. Создание туннеля

Проходим регистрацию на сайте https://tunnelbroker.net/

Создаём обычный туннель `Create Regular Tunnel`

В поле `IPv4 Endpoint (Your side):` нужно указать внешний IPv4 адрес.
В `Available Tunnel Servers:` выбираем ближайший сервер, или используя `ping` выбираем сервер с наименьшим временем отклика.

После создания туннеля откроется страница с информацией о нём.

Конфигурируем MTU туннеля. На вкладке `Advanced` изменяем MTU на `значение вычисленное на предыдущем шаге - 20`.

Пример:
```
MTU для Internet = 1460
1460 - 20 = 1440
Задаём значение 1440
```

По умолчанию туннель имеет значение MTU = 1480 (1500 - 20)

#### 5. Конфигурация Mikrotik

##### Создание туннельного интерфейса

На вкладке `Example Configurations` выбираем пример конфигурации для Mikrotik - автоматически будут подставлены ip адреса туннеля.

Заменяем значения
- `local-address` На локальный адрес роутера - обычно это `192.168.88.1`,
- `mtu` - такое же значение которое задали для туннеля на предыдущем шаге.


Пример (в примере используются несуществующие ipv6 адреса):

```
/interface 6to4 add comment="Hurricane Electric IPv6 Tunnel Broker" disabled=no local-address=<LOCAL_IP_ADDR> mtu=<CALCULATED_MTU> name=sit1 remote-address=216.66.84.46
/ipv6 route add comment="" disabled=no distance=1 dst-address=2000::/3 gateway=2001:db8:1:fcc::1 scope=30 target-scope=10
/ipv6 address add address=2001:db8:2:fcc::2/64 advertise=no disabled=no eui-64=no interface=sit1
```


##### Выделение ipv6 адреса клиентам в локальной сети.

Подставляем значения
- `Routed /64` - ipv6 адрес, на странице с информацией о туннеле
- `MTU` - значение вычисленное на 3 шаге. Можно не устанавливать если MTU Discovery работает. 


```
/ipv6 address add address=<Routed /64> interface=bridge advertise=yes
/ipv6 nd set [ find default=yes ] advertise-dns=yes mtu=<MTU>
```

##### Конфигурация dns

tunnelbroker.net предоставляет свой dns сервер можно использовать его `2001:470:20::2`. Или например dns серверы от Cloudflare.

```
/ip dns set allow-remote-requests=yes servers=2606:4700:4700::1111,2606:4700:4700::1001,2001:470:20::2,1.1.1.1,1.0.0.1
```


#### 6. Проверяем что всё работает

`ping6 google.com`

https://test-ipv6.com/

http://ipv6-test.com/


#### 7. Обновление настроек туннеля при смене внешнего IPv4 адреса

Tunnelbroker предоставляет API для обновление IPv4 адреса туннеля со стороны пользователя.

Ниже скрипт в котором необходимо заменить следующие значения чтобы он работал

- `TUNEL_ID` - значение на вкладке `IPv6 Tunnel`
- `USER_ID` - значение на вкладке advanced
- `UPDATE_KEY` - значение на вкладке advanced
- `WAN_INTERFACE_NAME` имя интерфейса с внешним ipv4 адресом, в моём случае это L2TP интерфейс

```
/system script
add dont-require-permissions=no name=updatet owner=admin policy=read,write,policy,test source="# Update Hurricane Electric IP\
    v6 Tunnel Client IPv4 address\
    \n\
    \n:global wanip\
    \n\
    \n:local userid \"<USER_ID>\"\
    \n:local updatekey \"<UPDATE_KEY>\"\
    \n:local tunnelid \"<TUNEL_ID>\"\
    \n:local remotehostname \"ipv4.tunnelbroker.net\"\
    \n:local waninterface \"<WAN_INTERFACE_NAME>\"\
    \n\
    \n:local outputfile \"tunnel-update-api-output\"\
    \n\
    \n:local currentip [/ip address get [/ip address find interface=\$waninterface] address ]\
    \n\
    \n:put \$wanip\
    \n:put \$currentip\
    \n:if ( \$wanip != \$currentip ) do={\
    \n\
    \n:set \$wanip \$currentip\
    \n\
    \n:log info \"Updating IPv6 Tunnel\"\
    \n\
    \n/tool fetch mode=https \\\
    \n                  host=(\$remotehostname) \\\
    \n                  url=\"https://\$userid:\$updatekey@ipv4.tunnelbroker.net/nic/update\?hostname=\$tunnelid\" \\\
    \n                  dst-path=(\$outputfile)\
    \n\
    \n/file remove \$outputfile\
    \n\
    \n}\
    \n\
    \n"
```


И создаём расписание для выполнения скрипта

```
/system scheduler
add interval=10s name=trigger on-event=updatet policy=read,write,policy,test start-date=feb/1/2019 start-time=19:00:00
```
