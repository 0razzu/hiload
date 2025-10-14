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
