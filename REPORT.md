# Отчёт по домашней работе

> Заполняйте отчёт по ходу выполнения задания. Для каждой найденной проблемы скопируйте секцию ниже и опишите не только внесённое изменение, но и ход диагностики.

## Проблема 1

### Что было обнаружено

При попытке подготовить окружение с помощью комады `sudo ./scripts/setup.sh` появилась ошибка `sudo: ./scripts/setup.sh: command not found`.


### Как проводилась диагностика

Я проверила права у файла `setup.sh`:

```bash
ls -l ./scripts/setup.sh
```

Вывод был следующий: `-rw-rw-r-- 1 vmuser vmuser 4596 Sep 29 14:04 ./scripts/setup.sh`

### В чём была причина

У **setup.sh** отсутствовали права на исполнение (флаг `x`) файла. 


### Что было изменено

Были выданы права для владельца/группы на исполнение:

```bash
chmod +x ./scripts/setup.sh
```

### Почему было выбрано это решение

Выдача прав на исполнение позволяет запускать скрипты (например, у которых расширение .sh), поэтому если система пишет об отсутствии команды, хотя скрипт должен быть исполняемым, проблема именно в правах на исполнение.

### Как проверялся результат

Я совершила повторный запуск команды:

```bash
sudo ./scripts/setup.sh
```

В итоге скрипт был успешно выполнен.


Так же при выполнении команды `sudo ./scripts/check.sh` возникла абсолютно аналогичная ошибка, которая была решена таким же образом через выдачу прав на исполнение файлу **check.sh**

---

## Проблема 2

### Что было обнаружено

При выполнении первой команды для диагности `systemctl status homework-app.service` была обнаружена ошибка в Process: `12507 ExecStart=/opt/linux-devops-homework/app.py (code=exited, status=203/EXEC)`.

### Как проводилась диагностика

Для диагностики данной проблемы я решила проверить содержимое unit-файла:

```bash
systemctl cat homework-app.service
```

В нем я обратила внимание на ExecStart: `ExecStarнt=/opt/linux-devops-homework/app.py`. Затем я решила проверить наличие файла **app.py** или наличие папки **app** в каталоге, который указан в ExecStart:

```bash
ls -l /opt/linux-devops-homework /opt/linux-devops-homework/app
```

В выводе был файл **server.py** и **No such file or directory**.

### В чём была причина

В unit-файле в ExecStart был указан неверный путь к запускаемому файлу, из за чего процесс было невозможно запустить. 

### Что было изменено

Для устранения данной проблемы я изменила путь до файла **server.py** в ExecStart в папке **systemd/homework-app.service** с помощью редактора nano: 

```bash
ExecStart=/opt/linux-devops-homework/app/server.py
```

Далее я создала новую директорию и переместила туда запускаемый файл:

```bash
sudo mkdir /opt/linux-devops-homework/app
sudo mv /opt/linux-devops-homework/server.py /opt/linux-devops-homework/app
```

После я скопировала файл из репозитория в системную директорию, чтобы были изменения для systemd и перезапустила daemon и сервис:

```bash
sudo cp systemd/homework-app.service /etc/systemd/system/homework-app.service
sudo systemctl daemon-reload
sudo systemctl start homework-app.service
```

### Почему было выбрано это решение

Код ошибки `status=203/EXEC` говорит о том, что systemd не смогла запустить процесс указанный в ExecStart, поэтому с указанием корректного пути к файлу приложение должно запускаться systemd.

### Как проверялся результат

Для проверки была запущена команда для отслеживания статуса:

```bash
systemctl status homework-app.service
```

В результате проверки ошибка `(code=exited, status=203/EXEC)` исчезла.

---

## Проблема 3

### Что было обнаружено

После исправления ExecStart в systemd при повторной проверке статуса сервиса возникла новая ошибка в Process: `(code=exited, status=1/FAILURE)`, а в Active: `failed (Result: exit-code)`.

### Как проводилась диагностика

Для поиска причины возникновения ошибки я посмотрела логи сервиса:

```bash
journalctl -u homework-app -n 30 --no-pager
```

В них приложение падало с ошибкой: `PermissionError: [Errno 13] Permission denied: '/var/lib/linux-devops-homework/startup.log'`

Далее я проверила владельца и права доступа проблемной директории:

```bash
ls -ld /var/lib/linux-devops-homework
```

В выводе получила следующее: `drwx------ 2 root root 4096 Sep 29 15:15 /var/lib/linux-devops-homework`

Далее я посмотрела от какого пользователя выполняется процесс сервиса:

```bash
sudo ls -l /proc/93332/status
```

По итогу в выводе оказался homework

### В чём была причина

Сервис должен выполняться от пользователя homework, но сама директория принадлежит root, поэтому процесс не мог создать файл **starup.log**.


### Что было изменено

Для решения возникшей проблемы потребовалось изменить владельца и права доступа к директории:

```bash
sudo chown -R homework:homework /var/lib/linux-devops-homework
sudo chmod 750 /var/lib/linux-devops-homework
```

### Почему было выбрано это решение

Код ошибки `(code=exited, status=1/FAILURE)` говорит о том, что процесс запустился, но завершился с ошибкой. При просмотре логов процесса было обнаружено, что ему недостаточно прав, поэтому потребовалось изменить их и передать владение пользователю homework, так как сервис выполняется из под него.

### Как проверялся результат

Для проверки результата сервис был перезапущен и проверен его статус:

```bash
sudo systemctl restart homework-app.service
systemctl status homework-app.service
```

В результате ошибка `(code=exited, status=1/FAILURE)` исчезла и сервис успешно запустился без ошибок: `Active: active (running) since Tue 2026-09-29 21:56:49 +05; 3s ago`

---

## Проблема 4

### Что было обнаружено

После изменения прав и владельцев сервис был успешно запущен, однако слушал он loopback интерфейс, что отразилось в статусе: `homework-app: listening on 127.0.0.1:8080`


### Как проводилась диагностика

Я посмотрела информацию о сетевых сокетах:

```bash
ss -lntp
```

В результате обнаружила такой вывод для :8080 порта: `LISTEN 0 5 127.0.0.1:8080 0.0.0.0:*`.

Далее проверила файл **config/app.conf**:

```bash
cat config/app.conf
```

В выводе можно было видеть, что host равен localhost.

### В чём была причина

В файле **config/app.conf** был неверно указан хост.

### Что было изменено

Был изменен хост в файле **config/app.conf** с помощью редактора nano:

```bash
host=0.0.0.0
```

### Почему было выбрано это решение

Адрес 0.0.0.0 позволяет приложению принимать входящие TCP-соединения на всех доступных сетевых интерфейсах системы.


### Как проверялся результат

Результат проверялся повторным запуском команды:

```bash
ss -lntp
```

Данный запуск показал следующее: `LISTEN 0 5 0.0.0.0:8080 0.0.0.0:*`

---

## Дополнительные наблюдения

При необходимости зафиксируйте здесь сведения о процессе, `/proc`, маршрутизации или другие результаты исследования системы, которые не относятся только к одной проблеме.


## Итоговый ход диагностики

При настройке окружения возникла ошибка `command not found`, которая была исправлена путем добавления файлам **setup.sh** и **check.sh** прав на исполнение. Далее при запуске первой команды `systemctl status homework-app.service` возникла ошибка  `(code=exited, status=203/EXEC)`, которая устранилась с помощью указания корректного пути в ExecStart в файле **systemd/homework-app.service**. После повторной проверки статуса сервиса обнаружилась проблема с доступами `(code=exited, status=1/FAILURE)`, которую удалось решить путем изменения владельца и прав к директории. После данного действия сервис успешно запускался, однако сетевые настройки были неудовлетворительны - хост в файле **config/app.conf** был равен localhost, что не позволяло слушать внешние интерфейсы, что было исправлено заменой хоста на 0.0.0.0. После сетевых изменений все проверки успешно прошли.


## Итоговая проверка

Вставьте полный вывод успешного запуска:

```bash
sudo ./scripts/check.sh
```

```text
Linux DevOps Homework Checker

[PASS] systemd ExecStart points to the application
[PASS] service runs as homework:homework
[PASS] state directory ownership and permissions are correct
[PASS] service is enabled
[PASS] service is active
[PASS] running process has the expected UID
[PASS] TCP/8080 listens on 0.0.0.0
[PASS] GET /health returns the expected response
[PASS] fixed systemd unit is saved in the repository
[PASS] fixed application configuration is saved in the repository

Result: 10 passed, 0 failed
```
