# WEB. High Performance WEB и CDN

## Задание 1. Создание Server Pool для backend

Создаю каталоги для серверов:
```
dmitry@host1:~$ mkdir backend1 && echo '<h1>Response from Backend Server 1</h1>' > backend1/index.html
dmitry@host1:~$ mkdir backend2 && echo '<h2>*** Response from Backend Server 2 ***</h2>' > backend2/index.html
dmitry@host1:~$ tree
.
├── backend1
│   └── index.html
├── backend2
│   └── index.html
...
```

Запускаю серверы:
```
dmitry@host1:~$ cd backend1
dmitry@host1:~/backend1$ python3 -m http.server 8081
Serving HTTP on 0.0.0.0 port 8081 (http://0.0.0.0:8081/) ...
```
```
dmitry@host1:~$ cd backend2
dmitry@host1:~/backend2$ python3 -m http.server 8082
Serving HTTP on 0.0.0.0 port 8082 (http://0.0.0.0:8082/) ...
```

Проверяю, что открываются:
![](1.1.png)
![](1.2.png)


## Задание 2. DNS Load Balancing с помощью dnsmasq

Добавляю строчки в `/etc/dnsmasq.conf`:
```
address=/my-awesome-highload-app.local/127.0.0.1
address=/my-awesome-highload-app.local/127.0.0.2
server=8.8.8.8
```

Они означают, что при запросе `my-awesome-highload-app.local` dnsmasq предложит сходить по какому-то из двух адресов, а всё, что dnsmasq не может обработать, (всё остальное) он отправит DNS’у Google.

С некоторыми плясками перезапускаю dnsmasq:
```
dmitry@host1:~$ sudo systemctl restart dnsmasq
Job for dnsmasq.service failed because the control process exited with error code.
See "systemctl status dnsmasq.service" and "journalctl -xeu dnsmasq.service" for details.
dmitry@host1:~$ systemctl status dnsmasq.service
× dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server
     Loaded: loaded (/usr/lib/systemd/system/dnsmasq.service; enabled; preset: enabled)
     Active: failed (Result: exit-code) since Mon 2025-10-27 21:08:32 UTC; 7s ago
    Process: 10723 ExecStartPre=/usr/share/dnsmasq/systemd-helper checkconfig (code=exited, status=0/SUCCESS)
    Process: 10728 ExecStart=/usr/share/dnsmasq/systemd-helper exec (code=exited, status=2)
        CPU: 17ms

Oct 27 21:08:32 host1 systemd[1]: Starting dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server...
Oct 27 21:08:32 host1 systemd-helper[10728]: dnsmasq: failed to create listening socket for port 53: Address already in use
Oct 27 21:08:32 host1 dnsmasq[10728]: failed to create listening socket for port 53: Address already in use
Oct 27 21:08:32 host1 systemd[1]: dnsmasq.service: Control process exited, code=exited, status=2/INVALIDARGUMENT
Oct 27 21:08:32 host1 dnsmasq[10728]: FAILED to start up
Oct 27 21:08:32 host1 systemd[1]: dnsmasq.service: Failed with result 'exit-code'.
Oct 27 21:08:32 host1 systemd[1]: Failed to start dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server.
dmitry@host1:~$ sudo lsof -i :53
COMMAND   PID            USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
systemd-r 666 systemd-resolve   14u  IPv4   8091      0t0  UDP 127.0.0.53:domain 
systemd-r 666 systemd-resolve   15u  IPv4   8092      0t0  TCP 127.0.0.53:domain (LISTEN)
systemd-r 666 systemd-resolve   16u  IPv4   8093      0t0  UDP 127.0.0.54:domain 
systemd-r 666 systemd-resolve   17u  IPv4   8094      0t0  TCP 127.0.0.54:domain (LISTEN)
dmitry@host1:~$ sudo systemctl stop systemd-resolved
dmitry@host1:~$ sudo nano /etc/resolv.conf
dmitry@host1:~$ systemctl status dnsmasq.service
● dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server
     Loaded: loaded (/usr/lib/systemd/system/dnsmasq.service; enabled; preset: enabled)
     Active: active (running) since Mon 2025-10-27 21:17:29 UTC; 2min 46s ago
    Process: 10791 ExecStartPre=/usr/share/dnsmasq/systemd-helper checkconfig (code=exited, status=0/SUCCESS)
    Process: 10796 ExecStart=/usr/share/dnsmasq/systemd-helper exec (code=exited, status=0/SUCCESS)
    Process: 10803 ExecStartPost=/usr/share/dnsmasq/systemd-helper start-resolvconf (code=exited, status=0/SUCCESS)
   Main PID: 10802 (dnsmasq)
      Tasks: 1 (limit: 4544)
     Memory: 928.0K (peak: 2.4M)
        CPU: 31ms
     CGroup: /system.slice/dnsmasq.service
             └─10802 /usr/sbin/dnsmasq -x /run/dnsmasq/dnsmasq.pid -u dnsmasq -r /run/dnsmasq/resolv.conf -7 /etc/dnsmasq.d,.dpkg-dist,.dpkg-old,.dpkg-new --local-service --trust-anchor=.,20326,8,2,E06D44B80B8F1D>

Oct 27 21:17:29 host1 systemd[1]: Starting dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server...
Oct 27 21:17:29 host1 dnsmasq[10802]: started, version 2.90 cachesize 150
Oct 27 21:17:29 host1 dnsmasq[10802]: DNS service limited to local subnets
Oct 27 21:17:29 host1 dnsmasq[10802]: compile time options: IPv6 GNU-getopt DBus no-UBus i18n IDN2 DHCP DHCPv6 no-Lua TFTP conntrack ipset nftset auth cryptohash DNSSEC loop-detect inotify dumpfile
Oct 27 21:17:29 host1 dnsmasq[10802]: read /etc/hosts - 8 names
Oct 27 21:17:29 host1 resolvconf[10810]: Dropped protocol specifier '.dnsmasq' from 'lo.dnsmasq'. Using 'lo' (ifindex=1).
Oct 27 21:17:29 host1 resolvconf[10810]: Failed to set DNS configuration: Link lo is loopback device.
Oct 27 21:17:29 host1 systemd[1]: Started dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server.
```

(В `/etc/resolv.conf` поменял адрес в `nameserver 127.0.0.53` на `127.0.0.1`.)

 Смотрю, что говорит `dig`:
 ```
 dmitry@host1:~$ dig my-awesome-highload-app.local

; <<>> DiG 9.18.39-0ubuntu0.24.04.2-Ubuntu <<>> my-awesome-highload-app.local
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 38846
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;my-awesome-highload-app.local.    IN    A

;; ANSWER SECTION:
my-awesome-highload-app.local. 0 IN    A    127.0.0.2
my-awesome-highload-app.local. 0 IN    A    127.0.0.1

;; Query time: 1 msec
;; SERVER: 127.0.0.1#53(127.0.0.1) (UDP)
;; WHEN: Mon Oct 27 21:34:53 UTC 2025
;; MSG SIZE  rcvd: 90

dmitry@host1:~$ dig google.com

; <<>> DiG 9.18.39-0ubuntu0.24.04.2-Ubuntu <<>> google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 2522
;; flags: qr rd ra; QUERY: 1, ANSWER: 6, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;google.com.            IN    A

;; ANSWER SECTION:
google.com.        140    IN    A    64.233.164.101
google.com.        140    IN    A    64.233.164.102
google.com.        140    IN    A    64.233.164.139
google.com.        140    IN    A    64.233.164.138
google.com.        140    IN    A    64.233.164.100
google.com.        140    IN    A    64.233.164.113

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1) (UDP)
;; WHEN: Mon Oct 27 21:35:02 UTC 2025
;; MSG SIZE  rcvd: 135
```

Всё резолвится. Если один из серверов у меня завалится, то ничего не помешает dnsmasq вернуть те же адреса (что пробили — то и отдаёт). Клиенты, которые попытаются сходить по адресу мёртвого сервера, постучатся в оба ip и два раза погрустят. Остановил второй бэк и стучусь (для сравнения сначала в первый):
```
dmitry@host1:~$ curl -v my-awesome-highload-app.local:8081
* Host my-awesome-highload-app.local:8081 was resolved.
* IPv6: (none)
* IPv4: 127.0.0.1, 127.0.0.2
*   Trying 127.0.0.1:8081...
* Connected to my-awesome-highload-app.local (127.0.0.1) port 8081
> GET / HTTP/1.1
> Host: my-awesome-highload-app.local:8081
> User-Agent: curl/8.5.0
> Accept: */*
> 
* HTTP 1.0, assume close after body
< HTTP/1.0 200 OK
< Server: SimpleHTTP/0.6 Python/3.12.3
< Date: Mon, 27 Oct 2025 21:52:21 GMT
< Content-type: text/html
< Content-Length: 40
< Last-Modified: Mon, 27 Oct 2025 17:01:57 GMT
< 
<h1>Response from Backend Server 1</h1>
* Closing connection
dmitry@host1:~$ curl -v my-awesome-highload-app.local:8082
* Host my-awesome-highload-app.local:8082 was resolved.
* IPv6: (none)
* IPv4: 127.0.0.1, 127.0.0.2
*   Trying 127.0.0.1:8082...
* connect to 127.0.0.1 port 8082 from 127.0.0.1 port 35822 failed: Connection refused
*   Trying 127.0.0.2:8082...
* connect to 127.0.0.2 port 8082 from 127.0.0.1 port 57820 failed: Connection refused
* Failed to connect to my-awesome-highload-app.local port 8082 after 2 ms: Couldn't connect to server
* Closing connection
curl: (7) Failed to connect to my-awesome-highload-app.local port 8082 after 2 ms: Couldn't connect to server
```


## Задание 3. Балансировка Layer 4 с  с помощью IPVS

Создаю `dummy1`:
```
dmitry@host1:~$ sudo ip link add dummy1 type dummy
dmitry@host1:~$ sudo ip a add 192.168.100.1/32 dev dummy1
dmitry@host1:~$ sudo ip link set dummy1 up
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
       valid_lft 42843sec preferred_lft 42843sec
    inet6 fe80::a00:27ff:fe9e:c1e4/64 scope link 
       valid_lft forever preferred_lft forever
3: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:d3:b3:e5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.57.2/24 brd 192.168.57.255 scope global enp0s9
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fed3:b3e5/64 scope link 
       valid_lft forever preferred_lft forever
4: dummy1: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 1a:68:67:31:03:62 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.1/32 scope global dummy1
       valid_lft forever preferred_lft forever
    inet6 fe80::1868:67ff:fe31:362/64 scope link 
       valid_lft forever preferred_lft forever
```

Создаю VS:
```
dmitry@host1:~$ sudo ipvsadm -A -t 192.168.100.1:80 -s rr
dmitry@host1:~$ sudo ipvsadm -a -t 192.168.100.1:80 -r 127.0.0.1:8081 -m
dmitry@host1:~$ sudo ipvsadm -a -t 192.168.100.1:80 -r 127.0.0.1:8082 -m
dmitry@host1:~$ sudo ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.100.1:80 rr
  -> 127.0.0.1:8081               Masq    1      0          0         
  -> 127.0.0.1:8082               Masq    1      0          0 
```

Проверяю балансировку. Должны по очереди возвращаться `:8081` и `:8082`, и счётчики должны увеличиваться равномерно:
```
dmitry@host1:~$ curl http://192.168.100.1
<h2>*** Response from Backend Server 2 ***</h2>
dmitry@host1:~$ sudo ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.100.1:80 rr
  -> 127.0.0.1:8081               Masq    1      0          0         
  -> 127.0.0.1:8082               Masq    1      0          1         
dmitry@host1:~$ curl http://192.168.100.1
<h1>Response from Backend Server 1</h1>
dmitry@host1:~$ sudo ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.100.1:80 rr
  -> 127.0.0.1:8081               Masq    1      0          1         
  -> 127.0.0.1:8082               Masq    1      0          1
dmitry@host1:~$ for i in {0..99}
> do
>     curl http://192.168.100.1
> done
<h2>*** Response from Backend Server 2 ***</h2>
<h1>Response from Backend Server 1</h1>
...
dmitry@host1:~$ sudo ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.100.1:80 rr
  -> 127.0.0.1:8081               Masq    1      0          51        
  -> 127.0.0.1:8082               Masq    1      0          51
```


## Задание 4. Балансировка L7 с помощью NGINX

Создаю файлик `/etc/nginx/conf.d/hiload.conf` со следующим содержимым:
```
upstream backend {
    server 127.0.0.1:8081 max_fails=7 fail_timeout=10s;
    server 127.0.0.1:8082 backup;
}

server {
    listen 127.0.0.1:8888;

    location / {
        proxy_pass http://backend;
        proxy_set_header X-high-load-test 123;
        proxy_connect_timeout 1s;
        proxy_send_timeout 1s;
        proxy_read_timeout 1s;
        send_timeout 1s;
    }
}
```

Проверяю, рестарчу:
```
dmitry@host1:~$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
dmitry@host1:~$ sudo systemctl restart nginx
dmitry@host1:~$ systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Tue 2025-10-28 19:55:55 UTC; 11s ago
       Docs: man:nginx(8)
    Process: 13723 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 13725 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 13727 (nginx)
      Tasks: 5 (limit: 4544)
     Memory: 3.6M (peak: 4.1M)
        CPU: 11ms
     CGroup: /system.slice/nginx.service
             ├─13727 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             ├─13728 "nginx: worker process"
             ├─13729 "nginx: worker process"
             ├─13730 "nginx: worker process"
             └─13731 "nginx: worker process"

Oct 28 19:55:55 host1 systemd[1]: Starting nginx.service - A high performance web server and a reverse proxy server...
Oct 28 19:55:55 host1 systemd[1]: Started nginx.service - A high performance web server and a reverse proxy server.
```

Стучусь в `:8888`:
```
dmitry@host1:~$ curl -v http://127.0.0.1:8888
*   Trying 127.0.0.1:8888...
* Connected to 127.0.0.1 (127.0.0.1) port 8888
> GET / HTTP/1.1
> Host: 127.0.0.1:8888
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.24.0 (Ubuntu)
< Date: Tue, 28 Oct 2025 19:58:01 GMT
< Content-Type: text/html
< Content-Length: 40
< Connection: keep-alive
< Last-Modified: Mon, 27 Oct 2025 17:01:57 GMT
< 
<h1>Response from Backend Server 1</h1>
* Connection #0 to host 127.0.0.1 left intact
```

И с tshark:
```
dmitry@host1:~$ sudo tshark -i lo -f "tcp port 8888 or tcp port 8081 or tcp port 8082" -Y "http" -E header=y -T fields -e frame.number -e tcp.srcport -e tcp.dstport -e http.request.method -e http.host -e http.request.uri -e http.user_agent -e http.response.code -e http.response.phrase
Running as user "root" and group "root". This could be dangerous.
frame.number    tcp.srcport    tcp.dstport    http.request.method    http.host    http.request.uri    http.user_agent    http.response.code    http.response.phrase
Capturing on 'Loopback: lo'
4    42092    8888    GET    127.0.0.1:8888    /    curl/8.5.0        
9    57044    8081    GET    backend    /    curl/8.5.0        
13    8081    57044                    200    OK
18    8888    42092                    200    OK
```

Вывода кастомных хэдеров в таком формате от tshark фиг добьёшься, поэтому посмотрим вообще всё (большую часть простыни вырезал):
```
dmitry@host1:~$ sudo tshark -i lo -f "tcp port 8888 or tcp port 8081 or tcp port 8082" -Y "http" -V
Running as user "root" and group "root". This could be dangerous.
Capturing on 'Loopback: lo'
Frame 4: 143 bytes on wire (1144 bits), 143 bytes captured (1144 bits) on interface lo, id 0
...
    [Protocols in frame: eth:ethertype:ip:tcp:http]
Ethernet II, Src: 00:00:00_00:00:00 (00:00:00:00:00:00), Dst: 00:00:00_00:00:00 (00:00:00:00:00:00)
...
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
...
Transmission Control Protocol, Src Port: 55710, Dst Port: 8888, Seq: 1, Ack: 1, Len: 77
...
Hypertext Transfer Protocol
    GET / HTTP/1.1\r\n
        [Expert Info (Chat/Sequence): GET / HTTP/1.1\r\n]
            [GET / HTTP/1.1\r\n]
            [Severity level: Chat]
            [Group: Sequence]
        Request Method: GET
        Request URI: /
        Request Version: HTTP/1.1
    Host: 127.0.0.1:8888\r\n
    User-Agent: curl/8.5.0\r\n
    Accept: */*\r\n
    \r\n
    [Full request URI: http://127.0.0.1:8888/]
    [HTTP request 1/1]

Frame 9: 178 bytes on wire (1424 bits), 178 bytes captured (1424 bits) on interface lo, id 0
...
    [Protocols in frame: eth:ethertype:ip:tcp:http]
Ethernet II, Src: 00:00:00_00:00:00 (00:00:00:00:00:00), Dst: 00:00:00_00:00:00 (00:00:00:00:00:00)
...
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
...
Transmission Control Protocol, Src Port: 55396, Dst Port: 8081, Seq: 1, Ack: 1, Len: 112
...
Hypertext Transfer Protocol
    GET / HTTP/1.0\r\n
        [Expert Info (Chat/Sequence): GET / HTTP/1.0\r\n]
            [GET / HTTP/1.0\r\n]
            [Severity level: Chat]
            [Group: Sequence]
        Request Method: GET
        Request URI: /
        Request Version: HTTP/1.0
    X-high-load-test: 123\r\n
    Host: backend\r\n
    Connection: close\r\n
    User-Agent: curl/8.5.0\r\n
    Accept: */*\r\n
    \r\n
    [Full request URI: http://backend/]
    [HTTP request 1/1]

Frame 13: 106 bytes on wire (848 bits), 106 bytes captured (848 bits) on interface lo, id 0
...
    [Protocols in frame: eth:ethertype:ip:tcp:http:data-text-lines]
Ethernet II, Src: 00:00:00_00:00:00 (00:00:00:00:00:00), Dst: 00:00:00_00:00:00 (00:00:00:00:00:00)
...
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
...
Transmission Control Protocol, Src Port: 8081, Dst Port: 55396, Seq: 186, Ack: 113, Len: 40
...
[2 Reassembled TCP Segments (225 bytes): #11(185), #13(40)]
    [Frame: 11, payload: 0-184 (185 bytes)]
    [Frame: 13, payload: 185-224 (40 bytes)]
...
Hypertext Transfer Protocol
    HTTP/1.0 200 OK\r\n
        [Expert Info (Chat/Sequence): HTTP/1.0 200 OK\r\n]
            [HTTP/1.0 200 OK\r\n]
            [Severity level: Chat]
            [Group: Sequence]
        Response Version: HTTP/1.0
        Status Code: 200
        [Status Code Description: OK]
        Response Phrase: OK
    Server: SimpleHTTP/0.6 Python/3.12.3\r\n
    Date: Tue, 28 Oct 2025 21:08:44 GMT\r\n
    Content-type: text/html\r\n
    Content-Length: 40\r\n
        [Content length: 40]
    Last-Modified: Mon, 27 Oct 2025 17:01:57 GMT\r\n
    \r\n
    [HTTP response 1/1]
    [Time since request: 0.003965714 seconds]
    [Request in frame: 9]
    [Request URI: http://backend/]
    File Data: 40 bytes
Line-based text data: text/html (1 lines)
    <h1>Response from Backend Server 1</h1>\n

Frame 19: 308 bytes on wire (2464 bits), 308 bytes captured (2464 bits) on interface lo, id 0
...
    [Protocols in frame: eth:ethertype:ip:tcp:http:data-text-lines]
Ethernet II, Src: 00:00:00_00:00:00 (00:00:00:00:00:00), Dst: 00:00:00_00:00:00 (00:00:00:00:00:00)
...
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
...
Transmission Control Protocol, Src Port: 8888, Dst Port: 55710, Seq: 1, Ack: 78, Len: 242
...
Hypertext Transfer Protocol
    HTTP/1.1 200 OK\r\n
        [Expert Info (Chat/Sequence): HTTP/1.1 200 OK\r\n]
            [HTTP/1.1 200 OK\r\n]
            [Severity level: Chat]
            [Group: Sequence]
        Response Version: HTTP/1.1
        Status Code: 200
        [Status Code Description: OK]
        Response Phrase: OK
    Server: nginx/1.24.0 (Ubuntu)\r\n
    Date: Tue, 28 Oct 2025 21:08:44 GMT\r\n
    Content-Type: text/html\r\n
    Content-Length: 40\r\n
        [Content length: 40]
    Connection: keep-alive\r\n
    Last-Modified: Mon, 27 Oct 2025 17:01:57 GMT\r\n
    \r\n
    [HTTP response 1/1]
    [Time since request: 0.004884049 seconds]
    [Request in frame: 4]
    [Request URI: http://127.0.0.1:8888/]
    File Data: 40 bytes
Line-based text data: text/html (1 lines)
    <h1>Response from Backend Server 1</h1>\n
```

В фрейме 9 хэдер стоит.

Теперь заменяю сервер на `127.0.0.1:8081` на такой тормозной:
```
import time
from http.server import BaseHTTPRequestHandler, HTTPServer

class SlowHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        time.sleep(1000)
        self.send_response(200)
        self.send_header("Content-type", "text/html")
        self.end_headers()
        self.wfile.write(b"<h1>Response from Backend Server 1</h1>")

def run(server_class=HTTPServer, handler_class=SlowHandler):
    server_address = ("127.0.0.1", 8081)
    httpd = server_class(server_address, handler_class)
    print("Starting slow server...")
    httpd.serve_forever()

if __name__ == "__main__":
    run()
```

И стучусь 10 раз:
```
dmitry@host1:~$ sudo tshark -i lo -f "tcp port 8888 or tcp port 8081 or tcp port 8082" -Y "http" -E header=y -T fields -e frame.number -e tcp.srcport -e tcp.dstport -e http.request.method -e http.host -e http.request.uri -e http.user_agent -e http.response.code -e http.response.phrase
Running as user "root" and group "root". This could be dangerous.
frame.number    tcp.srcport    tcp.dstport    http.request.method    http.host    http.request.uri    http.user_agent    http.response.code    http.response.phrase
Capturing on 'Loopback: lo'
12    58014    8888    GET    127.0.0.1:8888    /    curl/8.5.0        
17    59254    8081    GET    backend    /    curl/8.5.0        
23    40704    8082    GET    backend    /    curl/8.5.0        
27    8082    40704                    200    OK
32    8888    58014                    200    OK
40    58026    8888    GET    127.0.0.1:8888    /    curl/8.5.0        
45    59264    8081    GET    backend    /    curl/8.5.0        
52    40706    8082    GET    backend    /    curl/8.5.0        
56    8082    40706                    200    OK
61    8888    58026                    200    OK
69    58032    8888    GET    127.0.0.1:8888    /    curl/8.5.0        
74    59280    8081    GET    backend    /    curl/8.5.0        
81    40716    8082    GET    backend    /    curl/8.5.0        
85    8082    40716                    200    OK
90    8888    58032                    200    OK
98    58040    8888    GET    127.0.0.1:8888    /    curl/8.5.0        
103    59282    8081    GET    backend    /    curl/8.5.0        
110    40732    8082    GET    backend    /    curl/8.5.0        
114    8082    40732                    200    OK
119    8888    58040                    200    OK
127    58056    8888    GET    127.0.0.1:8888    /    curl/8.5.0        
132    59290    8081    GET    backend    /    curl/8.5.0        
139    47500    8082    GET    backend    /    curl/8.5.0        
143    8082    47500                    200    OK
148    8888    58056                    200    OK
156    35342    8888    GET    127.0.0.1:8888    /    curl/8.5.0        
161    33678    8081    GET    backend    /    curl/8.5.0        
168    47510    8082    GET    backend    /    curl/8.5.0        
172    8082    47510                    200    OK
177    8888    35342                    200    OK
185    35348    8888    GET    127.0.0.1:8888    /    curl/8.5.0        
190    33682    8081    GET    backend    /    curl/8.5.0        
197    47522    8082    GET    backend    /    curl/8.5.0        
201    8082    47522                    200    OK
207    8888    35348                    200    OK
215    35358    8888    GET    127.0.0.1:8888    /    curl/8.5.0        
222    47530    8082    GET    backend    /    curl/8.5.0        
226    8082    47530                    200    OK
231    8888    35358                    200    OK
239    35364    8888    GET    127.0.0.1:8888    /    curl/8.5.0        
245    47544    8082    GET    backend    /    curl/8.5.0        
249    8082    47544                    200    OK
254    8888    35364                    200    OK
262    35372    8888    GET    127.0.0.1:8888    /    curl/8.5.0        
268    47552    8082    GET    backend    /    curl/8.5.0        
272    8082    47552                    200    OK
277    8888    35372                    200    OK
```

Начиная с 8 раза nginx не пытается ходить в первый сервер.
