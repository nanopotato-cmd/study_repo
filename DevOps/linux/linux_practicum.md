# Linux Module
## Выполнение практического задания по восстановлению удалённого скрипта

В данной практической работе мною был инициализирован процесс через запуск скрипта `script.sh`, который ежесекундно записывает в `my.log` текущую дату. После чего данный скрипт был удалён, а его задержимое восстановлено через поиск `PID` процесса, далее просмотр дескриптора этого PID.

План выполнения задания:

1. Создание `script.sh` и `my.log` в `/opt/`
```bash
ubuntu@epd360ho13je925c34c0:/opt$ sudo touch script.sh
ubuntu@epd360ho13je925c34c0:/opt$ sudo touch my.log
```

2. Редактирование `script.sh` и его запуск в фоне, проверка работы скрипта
```bash
ubuntu@epd360ho13je925c34c0:/opt$ sudo nano script.sh
ubuntu@epd360ho13je925c34c0:/opt$ cat script.sh
while true; do
  date >> /opt/my.log
  sleep 1
done

ubuntu@epd360ho13je925c34c0:/opt$ sudo /opt/script.sh &
ubuntu@epd360ho13je925c34c0:/opt$ cat my.log
Tue Apr 28 11:27:25 UTC 2026
Tue Apr 28 11:27:26 UTC 2026
Tue Apr 28 11:27:27 UTC 2026
Tue Apr 28 11:27:28 UTC 2026
```

3. Удаление `script.sh`, убедились что в фоне процесс есть
```bash
ubuntu@epd360ho13je925c34c0:/opt$ sudo rm /opt/script.sh

ubuntu@epd360ho13je925c34c0:/opt$ ps aux | grep script.sh
root        1192  0.0  0.0   2800  1916 pts/1    S    11:27   0:00 sh /opt/script.sh
```

4. Нахождение исходного кода скрипта
```bash
ubuntu@epd360ho13je925c34c0:/opt$ ps aux | grep script.sh
root        1191  0.0  0.0  17132  2556 pts/1    Ss+  11:27   0:00 sudo /opt/script.sh
root        1192  0.0  0.0   2800  1916 pts/1    S    11:27   0:00 sh /opt/script.sh # <<< Нас интересует этот процесс (сам скрипт) >>>
ubuntu      1575  0.0  0.0   7084  2240 pts/0    S+   11:30   0:00 grep --color=auto script.sh

# По номеру PID находим номер файлового дескриптора (10)

ubuntu@epd360ho13je925c34c0:/opt$ sudo ls -l /proc/1192/fd/
total 0
lrwx------ 1 root root 64 Apr 28 11:28 0 -> /dev/pts/1
lrwx------ 1 root root 64 Apr 28 11:28 1 -> /dev/pts/1
lr-x------ 1 root root 64 Apr 28 11:28 10 -> '/opt/script.sh (deleted)' #<< Данный ФД >>
lrwx------ 1 root root 64 Apr 28 11:28 2 -> /dev/pts/1

# Прочитаем содержимое:

ubuntu@epd360ho13je925c34c0:/opt$ sudo cat /proc/1192/fd/10
while true; do
  date >> /opt/my.log
  sleep 1
done
```

Задача выполнена! 