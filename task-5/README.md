# Сетевой стек

## Задание 1. Анализ состояний TCP-соединений

Запускаю:
```
dmitry@host1:~$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

Ищу слушающий TCP-сокет этого сервера (он тут первый):
```
dmitry@host1:~$ ss -tln
State    Recv-Q   Send-Q     Local Address:Port     Peer Address:Port  Process  
LISTEN   0        5                0.0.0.0:8080          0.0.0.0:*              
LISTEN   0        4096       127.0.0.53%lo:53            0.0.0.0:*              
LISTEN   0        4096          127.0.0.54:53            0.0.0.0:*              
LISTEN   0        4096             0.0.0.0:22            0.0.0.0:*              
LISTEN   0        4096                [::]:22               [::]:*            
```

Стучусь через curl (получим данные и сразу отключимся) и смотрю состояние сокетов:
```
dmitry@host1:~$ curl -v http://localhost:8080
* Host localhost:8080 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:8080...
* connect to ::1 port 8080 from ::1 port 52280 failed: Connection refused
*   Trying 127.0.0.1:8080...
* Connected to localhost (127.0.0.1) port 8080
> GET / HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/8.5.0
> Accept: */*
> 
* HTTP 1.0, assume close after body
< HTTP/1.0 200 OK
< Server: SimpleHTTP/0.6 Python/3.12.3
< Date: Tue, 14 Oct 2025 17:59:32 GMT
< Content-type: text/html; charset=utf-8
< Content-Length: 704
< 
<!DOCTYPE HTML>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href=".bash_history">.bash_history</a></li>
<li><a href=".bash_logout">.bash_logout</a></li>
<li><a href=".bashrc">.bashrc</a></li>
<li><a href=".cache/">.cache/</a></li>
<li><a href=".lesshst">.lesshst</a></li>
<li><a href=".profile">.profile</a></li>
<li><a href=".ssh/">.ssh/</a></li>
<li><a href=".sudo_as_admin_successful">.sudo_as_admin_successful</a></li>
<li><a href="homework_key">homework_key</a></li>
<li><a href="shm_creator">shm_creator</a></li>
<li><a href="shm_creator.c">shm_creator.c</a></li>
</ul>
<hr>
</body>
</html>
* Closing connection
dmitry@host1:~$ ss -tan
State      Recv-Q  Send-Q   Local Address:Port     Peer Address:Port   Process  
LISTEN     0       5              0.0.0.0:8080          0.0.0.0:*               
LISTEN     0       4096     127.0.0.53%lo:53            0.0.0.0:*               
LISTEN     0       4096        127.0.0.54:53            0.0.0.0:*               
LISTEN     0       4096           0.0.0.0:22            0.0.0.0:*               
TIME-WAIT  0       0            127.0.0.1:8080        127.0.0.1:43946           
ESTAB      0       0         192.168.57.2:22       192.168.57.1:65209           
ESTAB      0       0         192.168.57.2:22       192.168.57.1:54412           
LISTEN     0       4096              [::]:22               [::]:*               
```

Висит сокет в состоянии TIME-WAIT, который спустя две минуты пропадает. Взялся он тут после того, как мы закрыли соединение с сервером: при закрытии сокет переходит из ESTABLISHED в TIME-WAIT, чтобы избежать перемешивания старых пакетов, где-то притормозивших в сети, с новыми (от новых соединений). Если бы мы сразу удаляли информацию о бывшем сокете, то нам будто бы ничего не оставалось кроме того, чтобы реагировать на заплутавшие пакеты как на новое соединение.

Понятно, что если мы будем постоянно подключаться и отключаться с клиентов, то такие угасающие сокеты могут нормально так поднакопиться в это временное окно. При должном упорстве мы просто исчерпаем весь пул доступных портов, и будет не через что подключиться к серверу. Побороть это можно, например, слушая соединения, сидя не на одном IP-адресе, а на нескольких или (совсем похоже на костыль) разрывая соединения через reset, а не стандартный fin/fin-ack. Покрасивее будет переиспользовать сокеты (time-wait reuse + TCP timestamps). Тогда если прилетает пакет с таким же IP и портом, но с новым таймстемпом, то это новое соединение — воскрешаем сокет; если же таймстемп как у старого соединения, его можно отбрасывать.


## Задание 2. Динамическая маршрутизация с BIRD

Создаю service_0:
```
dmitry@host1:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:9e:c1:e4 brd ff:ff:ff:ff:ff:ff
    inet 192.168.31.161/24 metric 100 brd 192.168.31.255 scope global dynamic enp0s8
       valid_lft 41613sec preferred_lft 41613sec
    inet6 fe80::a00:27ff:fe9e:c1e4/64 scope link 
       valid_lft forever preferred_lft forever
3: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:d3:b3:e5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.57.2/24 brd 192.168.57.255 scope global enp0s9
       valid_lft forever preferred_lft forever
    inet6 fd52:8366:e2bb:4318:a00:27ff:fed3:b3e5/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 2591897sec preferred_lft 604697sec
    inet6 fe80::a00:27ff:fed3:b3e5/64 scope link 
       valid_lft forever preferred_lft forever
dmitry@host1:~$ sudo ip link add service_0 type dummy
dmitry@host1:~$ sudo ip addr add 192.168.14.88/32 dev service_0
dmitry@host1:~$ ip route show
default via 192.168.31.1 dev enp0s8 proto dhcp src 192.168.31.161 metric 100 
192.168.31.0/24 dev enp0s8 proto kernel scope link src 192.168.31.161 metric 100 
192.168.31.1 dev enp0s8 proto dhcp scope link src 192.168.31.161 metric 100 
192.168.57.0/24 dev enp0s9 proto kernel scope link src 192.168.57.2 
dmitry@host1:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:9e:c1:e4 brd ff:ff:ff:ff:ff:ff
    inet 192.168.31.161/24 metric 100 brd 192.168.31.255 scope global dynamic enp0s8
       valid_lft 41550sec preferred_lft 41550sec
    inet6 fe80::a00:27ff:fe9e:c1e4/64 scope link 
       valid_lft forever preferred_lft forever
3: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:d3:b3:e5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.57.2/24 brd 192.168.57.255 scope global enp0s9
       valid_lft forever preferred_lft forever
    inet6 fd52:8366:e2bb:4318:a00:27ff:fed3:b3e5/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 2591968sec preferred_lft 604768sec
    inet6 fe80::a00:27ff:fed3:b3e5/64 scope link 
       valid_lft forever preferred_lft forever
4: service_0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 5a:a2:22:e1:73:1d brd ff:ff:ff:ff:ff:ff
    inet 192.168.14.88/32 scope global service_0
       valid_lft forever preferred_lft forever
dmitry@host1:~$ sudo ip link set service_0 up
dmitry@host1:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:9e:c1:e4 brd ff:ff:ff:ff:ff:ff
    inet 192.168.31.161/24 metric 100 brd 192.168.31.255 scope global dynamic enp0s8
       valid_lft 41497sec preferred_lft 41497sec
    inet6 fe80::a00:27ff:fe9e:c1e4/64 scope link 
       valid_lft forever preferred_lft forever
3: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:d3:b3:e5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.57.2/24 brd 192.168.57.255 scope global enp0s9
       valid_lft forever preferred_lft forever
    inet6 fd52:8366:e2bb:4318:a00:27ff:fed3:b3e5/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 2591915sec preferred_lft 604715sec
    inet6 fe80::a00:27ff:fed3:b3e5/64 scope link 
       valid_lft forever preferred_lft forever
4: service_0: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 5a:a2:22:e1:73:1d brd ff:ff:ff:ff:ff:ff
    inet 192.168.14.88/32 scope global service_0
       valid_lft forever preferred_lft forever
    inet6 fe80::58a2:22ff:fee1:731d/64 scope link 
       valid_lft forever preferred_lft forever
```

Настраиваю анонсирование через BIRD. Прописываваю в /etc/bird/bird.conf: 
```
log syslog all;

protocol device {
    scan time 10;
}

protocol kernel {
    ipv4 {
        import all;
        export all;
    };
}

protocol direct {
    ipv4;
}

protocol static {
    ipv4;
    route 192.168.14.88/32 via "service_0";
}

filter service_n {
    if net ~ 192.168.14.0/24 && net.len = 32 && ifname ~ "service_*" then {
        accept;
    }
    reject;
}

protocol rip rip_on_enp0s8 {
    ipv4 {
        import all;
        export filter service_n;
    };
    interface "enp0s8" {
        version 2;
    };
}
```

Перезапускаю BIRD:
```
dmitry@host1:~$ sudo systemctl restart bird
dmitry@host1:~$ systemctl status bird.service
● bird.service - BIRD Internet Routing Daemon
     Loaded: loaded (/usr/lib/systemd/system/bird.service; enabled; preset: enabled)
     Active: active (running) since Wed 2025-10-15 19:24:39 UTC; 12min ago
    Process: 2650 ExecStartPre=/usr/lib/bird/prepare-environment (code=exited, status=0/SUCCESS)
    Process: 2657 ExecStartPre=/usr/sbin/bird -p (code=exited, status=0/SUCCESS)
   Main PID: 2659 (bird)
      Tasks: 1 (limit: 4546)
     Memory: 920.0K (peak: 1.6M)
        CPU: 62ms
     CGroup: /system.slice/bird.service
             └─2659 /usr/sbin/bird -f -u bird -g bird

Oct 15 19:24:39 host1 systemd[1]: Starting bird.service - BIRD Internet Routing Daemon...
Oct 15 19:24:39 host1 systemd[1]: Started bird.service - BIRD Internet Routing Daemon.
Oct 15 19:24:39 host1 (bird)[2659]: bird.service: Referenced but unset environment variable evaluates to an empty string: BIRD_ARGS
Oct 15 19:24:39 host1 bird[2659]: Chosen router ID 192.168.14.88 according to interface service_0
Oct 15 19:24:39 host1 bird[2659]: Started
```

Проверяю, что анонс идёт:
```
dmitry@host1:~$ sudo tcpdump -i enp0s8 port 520 -vv
tcpdump: listening on enp0s8, link-type EN10MB (Ethernet), snapshot length 262144 bytes
19:38:23.061134 IP (tos 0xc0, ttl 1, id 6348, offset 0, flags [none], proto UDP (17), length 52)
    host1.route > rip2-routers.mcast.net.route: [bad udp cksum 0xc084 -> 0x6a45!] 
    RIPv2, Response, length: 24, routes: 1 or less
      AFI IPv4,           host1/32, tag 0x0000, metric: 1, next-hop: self
    0x0000:  0202 0000 0002 0000 c0a8 0e58 ffff ffff
    0x0010:  0000 0000 0000 0001
19:38:53.061611 IP (tos 0xc0, ttl 1, id 6862, offset 0, flags [none], proto UDP (17), length 52)
    host1.route > rip2-routers.mcast.net.route: [bad udp cksum 0xc084 -> 0x6a45!] 
    RIPv2, Response, length: 24, routes: 1 or less
      AFI IPv4,           host1/32, tag 0x0000, metric: 1, next-hop: self
    0x0000:  0202 0000 0002 0000 c0a8 0e58 ffff ffff
    0x0010:  0000 0000 0000 0001
```

Рандомный онлайн-декодер сообщает, что тут вполне ожидаемые значения: `2 2 0 0 0 2 0 0 192 168 14 88 255 255 255 255 0 0 0 0 0 0 0 1` (видим адрес и маску).

Накидаем ещё интерфейсов:
```
dmitry@host1:~$ sudo ip link add service_1 type dummy && sudo ip addr add 192.168.14.1/30 dev service_1 && sudo ip link set service_1 up
dmitry@host1:~$ sudo ip link add service_2 type dummy && sudo ip addr add 192.168.10.4/32 dev service_2 && sudo ip link set service_2 up
dmitry@host1:~$ sudo ip link add srv_1 type dummy && sudo ip addr add 192.168.14.4/32 dev srv_1 && sudo ip link set srv_1 up
dmitry@host1:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:9e:c1:e4 brd ff:ff:ff:ff:ff:ff
    inet 192.168.31.161/24 metric 100 brd 192.168.31.255 scope global dynamic enp0s8
       valid_lft 25417sec preferred_lft 25417sec
    inet6 2a03:d000:201:b8e7:a00:27ff:fe9e:c1e4/64 scope global mngtmpaddr noprefixroute 
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe9e:c1e4/64 scope link 
       valid_lft forever preferred_lft forever
3: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:d3:b3:e5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.57.2/24 brd 192.168.57.255 scope global enp0s9
       valid_lft forever preferred_lft forever
    inet6 fd52:8366:e2bb:4318:a00:27ff:fed3:b3e5/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 2591914sec preferred_lft 604714sec
    inet6 fe80::a00:27ff:fed3:b3e5/64 scope link 
       valid_lft forever preferred_lft forever
4: service_0: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 5a:a2:22:e1:73:1d brd ff:ff:ff:ff:ff:ff
    inet 192.168.14.88/32 scope global service_0
       valid_lft forever preferred_lft forever
    inet6 fe80::58a2:22ff:fee1:731d/64 scope link 
       valid_lft forever preferred_lft forever
5: service_1: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether a2:e5:74:ba:86:aa brd ff:ff:ff:ff:ff:ff
    inet 192.168.14.1/30 scope global service_1
       valid_lft forever preferred_lft forever
    inet6 fe80::a0e5:74ff:feba:86aa/64 scope link 
       valid_lft forever preferred_lft forever
6: service_2: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether b2:a1:6c:de:20:95 brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.4/32 scope global service_2
       valid_lft forever preferred_lft forever
    inet6 fe80::b0a1:6cff:fede:2095/64 scope link 
       valid_lft forever preferred_lft forever
7: srv_1: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 4e:62:0c:64:00:bd brd ff:ff:ff:ff:ff:ff
    inet 192.168.14.4/32 scope global srv_1
       valid_lft forever preferred_lft forever
    inet6 fe80::4c62:cff:fe64:bd/64 scope link 
       valid_lft forever preferred_lft forever
```

Висевший всё это время `tcpdump` не поймал ничего лишнего:
```
19:53:10.277132 IP (tos 0xc0, ttl 1, id 29748, offset 0, flags [none], proto UDP (17), length 52)
    host1.route > rip2-routers.mcast.net.route: [bad udp cksum 0xc084 -> 0x6a45!] 
    RIPv2, Response, length: 24, routes: 1 or less
      AFI IPv4,           host1/32, tag 0x0000, metric: 1, next-hop: self
    0x0000:  0202 0000 0002 0000 c0a8 0e58 ffff ffff
    0x0010:  0000 0000 0000 0001
19:53:40.277593 IP (tos 0xc0, ttl 1, id 46271, offset 0, flags [none], proto UDP (17), length 52)
    host1.route > rip2-routers.mcast.net.route: [bad udp cksum 0xc084 -> 0x6a45!] 
    RIPv2, Response, length: 24, routes: 1 or less
      AFI IPv4,           host1/32, tag 0x0000, metric: 1, next-hop: self
    0x0000:  0202 0000 0002 0000 c0a8 0e58 ffff ffff
    0x0010:  0000 0000 0000 0001
```

Докинем интерфейс, который должен попасть в анонс:
```
dmitry@host1:~$ sudo ip link add service_3 type dummy && sudo ip addr add 192.168.14.12/32 dev service_3 && sudo ip link set service_3 up
dmitry@host1:~$ ip a
...
8: service_3: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 4a:e6:01:b4:6f:93 brd ff:ff:ff:ff:ff:ff
    inet 192.168.14.12/32 scope global service_3
       valid_lft forever preferred_lft forever
    inet6 fe80::48e6:1ff:feb4:6f93/64 scope link 
       valid_lft forever preferred_lft forever
```

Действительно:
```
19:55:02.707079 IP (tos 0xc0, ttl 1, id 25977, offset 0, flags [none], proto UDP (17), length 52)
    host1.route > rip2-routers.mcast.net.route: [bad udp cksum 0xc084 -> 0x6a91!] 
    RIPv2, Response, length: 24, routes: 1 or less
      AFI IPv4,           host1/32, tag 0x0000, metric: 1, next-hop: self
    0x0000:  0202 0000 0002 0000 c0a8 0e0c ffff ffff
    0x0010:  0000 0000 0000 0001
19:55:10.279369 IP (tos 0xc0, ttl 1, id 26748, offset 0, flags [none], proto UDP (17), length 72)
    host1.route > rip2-routers.mcast.net.route: [bad udp cksum 0xc098 -> 0x9b65!] 
    RIPv2, Response, length: 44, routes: 2 or less
      AFI IPv4,           host1/32, tag 0x0000, metric: 1, next-hop: self
      AFI IPv4,           host1/32, tag 0x0000, metric: 1, next-hop: self
    0x0000:  0202 0000 0002 0000 c0a8 0e0c ffff ffff
    0x0010:  0000 0000 0000 0001 0002 0000 c0a8 0e58
    0x0020:  ffff ffff 0000 0000 0000 0001
19:55:40.277431 IP (tos 0xc0, ttl 1, id 43886, offset 0, flags [none], proto UDP (17), length 72)
    host1.route > rip2-routers.mcast.net.route: [bad udp cksum 0xc098 -> 0x9b65!] 
    RIPv2, Response, length: 44, routes: 2 or less
      AFI IPv4,           host1/32, tag 0x0000, metric: 1, next-hop: self
      AFI IPv4,           host1/32, tag 0x0000, metric: 1, next-hop: self
    0x0000:  0202 0000 0002 0000 c0a8 0e0c ffff ffff
    0x0010:  0000 0000 0000 0001 0002 0000 c0a8 0e58
    0x0020:  ffff ffff 0000 0000 0000 0001
```

Перевожу цифры в цифры: `2 2 0 0 0 2 0 0 192 168 14 12 255 255 255 255 0 0 0 0 0 0 0 1 0 2 0 0 192 168 14 88 255 255 255 255 0 0 0 0 0 0 0 1` — и 192.168.14.12/32, и 192.168.14.88/32 на месте, и никто лишний не пробрался.


## Задание 3. Настройка фаервола

Смотрю, что сейчас понастроено:
```
dmitry@host1:~$ sudo iptables -L -n -v
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
dmitry@host1:~$ sudo nft list ruleset
```

Запускаю сервер:
```
dmitry@host1:~$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

Спокойно захожу, в т. ч. снаружи:
```
dmitry@host1:~$ curl localhost:8080
<!DOCTYPE HTML>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href=".bash_history">.bash_history</a></li>
<li><a href=".bash_logout">.bash_logout</a></li>
<li><a href=".bashrc">.bashrc</a></li>
<li><a href=".cache/">.cache/</a></li>
<li><a href=".lesshst">.lesshst</a></li>
<li><a href=".profile">.profile</a></li>
<li><a href=".ssh/">.ssh/</a></li>
<li><a href=".sudo_as_admin_successful">.sudo_as_admin_successful</a></li>
<li><a href="homework_key">homework_key</a></li>
<li><a href="shm_creator">shm_creator</a></li>
<li><a href="shm_creator.c">shm_creator.c</a></li>
</ul>
<hr>
</body>
</html>
```

```
MacBook-Pro-dmitry:~ dmitrypatoka$ curl host1:8080
<!DOCTYPE HTML>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href=".bash_history">.bash_history</a></li>
<li><a href=".bash_logout">.bash_logout</a></li>
<li><a href=".bashrc">.bashrc</a></li>
<li><a href=".cache/">.cache/</a></li>
<li><a href=".lesshst">.lesshst</a></li>
<li><a href=".profile">.profile</a></li>
<li><a href=".ssh/">.ssh/</a></li>
<li><a href=".sudo_as_admin_successful">.sudo_as_admin_successful</a></li>
<li><a href="homework_key">homework_key</a></li>
<li><a href="shm_creator">shm_creator</a></li>
<li><a href="shm_creator.c">shm_creator.c</a></li>
</ul>
<hr>
</body>
</html>
```

Логи пишутся:
```
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
127.0.0.1 - - [15/Oct/2025 20:08:32] "GET / HTTP/1.1" 200 -
192.168.57.1 - - [15/Oct/2025 20:09:51] "GET / HTTP/1.1" 200 -
```

Затыкаю, проверяю:
```
dmitry@host1:~$ sudo iptables -A INPUT -p tcp --dport 8080 -j REJECT
dmitry@host1:~$ sudo iptables -L -n -v
Chain INPUT (policy ACCEPT 385 packets, 26972 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 REJECT     6    --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 reject-with icmp-port-unreachable

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination  
```

Стучусь:
```
dmitry@host1:~$ curl localhost:8080
curl: (7) Failed to connect to localhost port 8080 after 0 ms: Couldn't connect to server
```

```
MacBook-Pro-dmitry:~ dmitrypatoka$ curl host1:8080
curl: (7) Failed to connect to host1 port 8080 after 3 ms: Couldn't connect to server
```
