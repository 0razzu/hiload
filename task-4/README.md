# Linux работа с памятью и процессами

## Задание 1. Systemd

Создаю файлик, устанавливаю нужные привилегии:
```
dmitry@host2:~$ sudo nano /usr/local/bin/homework_service.sh
dmitry@host2:~$ ls -l /usr/local/bin/homework_service.sh
-rw-r--r-- 1 root root 146 Oct  7 15:34 /usr/local/bin/homework_service.sh
dmitry@host2:~$ sudo chmod +x /usr/local/bin/homework_service.sh
dmitry@host2:~$ ls -l /usr/local/bin/homework_service.sh
-rwxr-xr-x 1 root root 146 Oct  7 15:34 /usr/local/bin/homework_service.sh
```

Создаю unit `/etc/systemd/system/homework.service` с таким содержимым:
```
[Unit]
Description=Homework Service

[Service]
ExecStart=/usr/local/bin/homework_service.sh
Restart=always
RestartSec=15

[Install]
WantedBy=multi-user.target
```

Заставляю systemd перечитать юниты и стартовать демон:
```
dmitry@host2:~$ sudo systemctl daemon-reload
dmitry@host2:~$ sudo systemctl start homework.service
```

Проверяю, что запущен:
```
dmitry@host2:~$ sudo systemctl status homework.service
dmitry@host2:~$ sudo systemctl status homework.service
● homework.service - Homework Service
     Loaded: loaded (/etc/systemd/system/homework.service; enabled; preset: enabled)
     Active: active (running) since Tue 2025-10-07 16:55:13 UTC; 2s ago
   Main PID: 2491 (homework_servic)
      Tasks: 2 (limit: 4614)
     Memory: 564.0K (peak: 800.0K)
        CPU: 36ms
     CGroup: /system.slice/homework.service
             ├─2491 /bin/bash /usr/local/bin/homework_service.sh
             └─2493 sleep 15

Oct 07 16:55:13 host2 systemd[1]: Started homework.service - Homework Service.
Oct 07 16:55:13 host2 homework_service.sh[2491]: My custom service has started.
```

Проверяю логи, раз в 15 с добавляется запись:
```
dmitry@host2:~$ tail -f /tmp/homework_service.log
Service heartbeat: Tue Oct  7 04:34:04 PM UTC 2025
Service heartbeat: Tue Oct  7 04:49:56 PM UTC 2025
Service heartbeat: Tue Oct  7 04:50:11 PM UTC 2025
Service heartbeat: Tue Oct  7 04:50:26 PM UTC 2025
Service heartbeat: Tue Oct  7 04:50:41 PM UTC 2025
Service heartbeat: Tue Oct  7 04:50:56 PM UTC 2025
Service heartbeat: Tue Oct  7 04:51:11 PM UTC 2025
Service heartbeat: Tue Oct  7 04:51:26 PM UTC 2025
Service heartbeat: Tue Oct  7 04:55:13 PM UTC 2025
Service heartbeat: Tue Oct  7 04:55:28 PM UTC 2025
Service heartbeat: Tue Oct  7 04:55:43 PM UTC 2025
Service heartbeat: Tue Oct  7 04:55:58 PM UTC 2025
^C
```

Добавляю в автозапуск:
```
dmitry@host2:~$ sudo systemctl enable homework.service
Created symlink /etc/systemd/system/multi-user.target.wants/homework.service → /etc/systemd/system/homework.service.
```

Грохаю сервис и проверяю рестарт:
```
dmitry@host2:~$ sudo kill 2491
dmitry@host2:~$ sudo systemctl status homework.service
● homework.service - Homework Service
     Loaded: loaded (/etc/systemd/system/homework.service; enabled; preset: enabled)
     Active: activating (auto-restart) since Tue 2025-10-07 16:56:15 UTC; 3s ago
    Process: 2491 ExecStart=/usr/local/bin/homework_service.sh (code=killed, signal=TERM)
   Main PID: 2491 (code=killed, signal=TERM)
        CPU: 256ms
dmitry@host2:~$ sudo systemctl status homework.service
● homework.service - Homework Service
     Loaded: loaded (/etc/systemd/system/homework.service; enabled; preset: enabled)
     Active: activating (auto-restart) since Tue 2025-10-07 16:56:15 UTC; 8s ago
    Process: 2491 ExecStart=/usr/local/bin/homework_service.sh (code=killed, signal=TERM)
   Main PID: 2491 (code=killed, signal=TERM)
        CPU: 256ms
dmitry@host2:~$ tail -f /tmp/homework_service.log
Service heartbeat: Tue Oct  7 04:50:41 PM UTC 2025
Service heartbeat: Tue Oct  7 04:50:56 PM UTC 2025
Service heartbeat: Tue Oct  7 04:51:11 PM UTC 2025
Service heartbeat: Tue Oct  7 04:51:26 PM UTC 2025
Service heartbeat: Tue Oct  7 04:55:13 PM UTC 2025
Service heartbeat: Tue Oct  7 04:55:28 PM UTC 2025
Service heartbeat: Tue Oct  7 04:55:43 PM UTC 2025
Service heartbeat: Tue Oct  7 04:55:58 PM UTC 2025
Service heartbeat: Tue Oct  7 04:56:13 PM UTC 2025
Service heartbeat: Tue Oct  7 04:56:30 PM UTC 2025
Service heartbeat: Tue Oct  7 04:56:45 PM UTC 2025
Service heartbeat: Tue Oct  7 04:57:00 PM UTC 2025
^C
```

Смотрю наиболее медленно стартующие сервисы:
```
dmitry@host2:~$ systemd-analyze blame | head -n 5
28.997s snapd.seeded.service
27.503s snapd.service
19.392s motd-news.service
15.288s homework.service
14.371s apport.service
```
