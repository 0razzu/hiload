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


## Задание 2. Межпроцессное взаимодействие (IPC) с разделяемой памятью

Скопипастил код и команды запуска из задания.

```
dmitry@dmitry:~$ ./shm_creator
Shared memory segment created.
ID: 1
Key: 0x41000011
Run 'ipcs -m' to see it. Process will exit in 60 seconds...
```

```
dmitry@dmitry:~$ ipcs -m

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status      
0x41000011 1          dmitry     666        1024       0   
```

По истечении 60 с табличка, естественно, опустевает. С первыми 5 столбцами как будто всё ожидаемо. Столбец `nattch` показывает, сколько процессов подключено к сегменту. Мы же не подключаемся, а просто создаём и ждём. Инкремент этого значения происходит при вызове `shmat` (*Upon successful completion, shmat() shall increment the value of shm_nattch in the data structure associated with the shared memory ID of the attached shared memory segment* — https://man7.org/linux/man-pages/man3/shmat.3p.html), декремент — по `shmdt` (*Upon successful completion, shmdt() shall decrement the value of shm_nattch in the data structure associated with the shared memory ID of the attached shared memory segment* — https://man7.org/linux/man-pages/man3/shmdt.3p.html). При создании по `shmget` инициализируем нулём (*When a new shared memory segment is created, its contents are initialized to zero values, and its associated data structure, hmid_ds (see shmctl(2)), is initialized as follows: ... shm_lpid, shm_nattch, shm_atime, and shm_dtime are set to 0.*).

Добавляю вызов shm_attach для теста:
```
...
    printf("ID: %d\nKey: 0x%x\n", shmid, key);
    sleep(15);
    shmat(shmid, NULL, 0);
    printf("Run 'ipcs -m' to see it. Process will exit in 60 seconds...\n");
...
```

Ожидаемо получаю 1 спустя 15 с:
```
dmitry@dmitry:~$ ipcs -m

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status      
0x41000011 2          dmitry     666        1024       0                       

dmitry@dmitry:~$ ipcs -m

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status      
0x41000011 2          dmitry     666        1024       1
```

Ещё есть колонка `status`. Мы никаких флагов при создании не ставили, поэтому она пустая. Если же, например, сотворить такое: `shmctl(shmid, SHM_LOCK, NULL);`, то получим:
```
dmitry@dmitry:~$ ipcs -m

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status      
0x41000011 3          dmitry     666        1024       1                 locked
```

Ещё можно заметить, что ключ не меняется (`ftok` работает детерминированно), а id инкрементится на каждом запуске, т. к. каждый раз просим ядро создать новый сегмент (`IPC_CREAT` — *Create a new segment.  If this flag is not used, then shmget() will find the segment associated with key and check to see if the user has permission to access the segment.* — https://www.man7.org/linux/man-pages/man2/shmget.2.html).


## Задание 3. Анализ памяти процессов (VSZ vs RSS)

```
dmitry@dmitry:~$ python3 -c "print('Allocating memory...'); a = 'X' * (250 * 1024 * 1024); import time; print('Memory allocated. Sleeping...'); time.sleep(120);"&
[1] 2890
dmitry@dmitry:~$ Allocating memory...
Memory allocated. Sleeping...
```

```
dmitry@dmitry:~$ ps -o pid,user,%mem,rss,vsz,comm -p 2890
    PID USER     %MEM   RSS    VSZ COMMAND
   2890 dmitry    6.6 265472 273268 python3
```

Если убрать команду `a = ...`:
```
dmitry@dmitry:~$ ps -o pid,user,%mem,rss,vsz,comm -p 2918
    PID USER     %MEM   RSS    VSZ COMMAND
   2918 dmitry    0.2  9472  17260 python3
```

Запрашивали 256.000 КБ под строчку. По RSS ровно такая разница. Здесь реальная память заиспользовалась потому, что мы не только сказали: «Система, дай нам кусок памяти», а ещё и записали в него что-то. И стек, и куча, естественно, должны где-то храниться. Без создания строки RSS ненулевой, т. к. Питон тоже не бесплатный и что-то пишет.

А то, что VSZ > RSS, — совершенно ожидаемое состояние, т. к. в виртуальной памяти держатся неинициализированные куски памяти, загруженные библиотеки и пр. 
