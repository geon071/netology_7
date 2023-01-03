# Домашнее задание к занятию "3.3. Операционные системы. Лекция 1"

## Какой системный вызов делает команда cd?

      vagrant@vagrant:~$ strace /bin/bash -c 'cd /tmp' 2>&1 | grep tmp
      execve("/bin/bash", ["/bin/bash", "-c", "cd /tmp"], 0x7ffe56ff2910 /* 23 vars */) = 0
      stat("/tmp", {st_mode=S_IFDIR|S_ISVTX|0777, st_size=4096, ...}) = 0
      chdir("/tmp")

На основании вывода получается - chdir("/tmp")

## Попробуйте использовать команду file на объекты разных типов в файловой системе. Используя strace выясните, где находится база данных file, на основании которой она делает свои догадки.

      vagrant@vagrant:~$ strace file /bin/bash

При выводе обращал внимание на системные вызовы - openat (открытие файлов)

      vagrant@vagrant:~$ strace file /bin/bash 2>&1 | grep openat
      openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
      openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libmagic.so.1", O_RDONLY|O_CLOEXEC) = 3
      openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
      openat(AT_FDCWD, "/lib/x86_64-linux-gnu/liblzma.so.5", O_RDONLY|O_CLOEXEC) = 3
      openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libbz2.so.1.0", O_RDONLY|O_CLOEXEC) = 3
      openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libz.so.1", O_RDONLY|O_CLOEXEC) = 3
      openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libpthread.so.0", O_RDONLY|O_CLOEXEC) = 3
      openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
      openat(AT_FDCWD, "/etc/magic.mgc", O_RDONLY) = -1 ENOENT (No such file or directory)
      openat(AT_FDCWD, "/etc/magic", O_RDONLY) = 3
      openat(AT_FDCWD, "/usr/share/misc/magic.mgc", O_RDONLY) = 3
      openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/gconv/gconv-modules.cache", O_RDONLY) = 3
      openat(AT_FDCWD, "/bin/bash", O_RDONLY|O_NONBLOCK) = 3

Из перечисленного на БД утилиты похоже:

      openat(AT_FDCWD, "/etc/magic.mgc", O_RDONLY) = -1 ENOENT (No such file or directory)
      openat(AT_FDCWD, "/etc/magic", O_RDONLY) = 3
      openat(AT_FDCWD, "/usr/share/misc/magic.mgc", O_RDONLY) = 3

## Предположим, приложение пишет лог в текстовый файл. Этот файл оказался удален (deleted в lsof), однако возможности сигналом сказать приложению переоткрыть файлы или просто перезапустить приложение – нет. Так как приложение продолжает писать в удаленный файл, место на диске постепенно заканчивается. Основываясь на знаниях о перенаправлении потоков предложите способ обнуления открытого удаленного файла (чтобы освободить место на файловой системе).

Воспроизвел ситуацию, запустив вывод команды top в файл:

     vagrant@vagrant:~$ top > test.txt

Далее удалил файл и проверил через утилиту lsof состояние дескриптора:

     vagrant@vagrant:~$ lsof -p 14797 | grep test
     top     14797 vagrant    1w   REG  253,0     6379    1310811 /home/vagrant/test.txt (deleted)
     
Очистить содержимое удаленного файла можно просто перенаправив вывод на файловый дескриптор, в данном случае получается дескриптор с номером 1

     vagrant@vagrant:~$ cat /dev/null >/proc/14797/fd/1

## Занимают ли зомби-процессы какие-то ресурсы в ОС (CPU, RAM, IO)?

Нет не занимают, они присутсвуют лишь как записи в таблице процессов.

## В iovisor BCC есть утилита opensnoop. На какие файлы вы увидели вызовы группы open за первую секунду работы утилиты?

При попытке запустить от пользователя получил ошибку:

     vagrant@vagrant:~$ opensnoop-bpfcc
     could not open bpf map: events, error: Operation not permitted
     Traceback (most recent call last):
       File "/usr/sbin/opensnoop-bpfcc", line 180, in <module>
         b = BPF(text=bpf_text)
       File "/usr/lib/python3/dist-packages/bcc/__init__.py", line 347, in __init__
         raise Exception("Failed to compile BPF module %s" % (src_file or "<text>"))
     Exception: Failed to compile BPF module <text>

Запустилось с sudo:

     vagrant@vagrant:~$ sudo opensnoop-bpfcc
     PID    COMM               FD ERR PATH
     1265   vminfo              4   0 /var/run/utmp
     628    dbus-daemon        -1   2 /usr/local/share/dbus-1/system-services
     628    dbus-daemon        20   0 /usr/share/dbus-1/system-services
     628    dbus-daemon        -1   2 /lib/dbus-1/system-services
     628    dbus-daemon        20   0 /var/lib/snapd/dbus-1/system-services/

## Какой системный вызов использует uname -a? Приведите цитату из man по этому системному вызову, где описывается альтернативное местоположение в /proc, где можно узнать версию ядра и релиз ОС.

На основании вывода strace получается системный вызов uname()

     vagrant@vagrant:~$ strace uname -a
     ...
     uname({sysname="Linux", nodename="vagrant", ...}) = 0
     fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0), ...}) = 0
     uname({sysname="Linux", nodename="vagrant", ...}) = 0
     uname({sysname="Linux", nodename="vagrant", ...}) = 0
     ...

По умолчанию в рекомендуемом образе ОС, в man не было справочной информации по системным вызовам, установил рекомендуемый пакет:

    vagrant@vagrant:~$ sudo apt install manpages-dev

И после нашел вывод в справке:

     vagrant@vagrant:~$ man 2 uname
     ...
            Part of the utsname information is also accessible via /proc/sys/kernel/{ostype, hostname, osrelease, version,
            domainname}.
     ...

## Чем отличается последовательность команд через ; и через && в bash? Есть ли смысл использовать в bash &&, если применить set -e?

; - разделяет команды для последовательного исполнения, && - условный оператор, если подправить первую команду, чтобы она не возвращала ошибку получится:

    vagrant@vagrant:~$ test -d /tmp && echo Hi
    Hi

Т.е. в случае если какая-то из команд возвращает ошибку, исполнение, при использовании оператора &&, прерывается.

"set -e" - мгновенно останавливает исполнение, в случае возвращения не нулевого результата исполнения любой команды в последовательности, кроме последней. Отвечая на вопрос, думаю нет, смысла не имеет.

## Из каких опций состоит режим bash set -euxo pipefail и почему его хорошо было бы использовать в сценариях?

- "-e" - мгновенно останавливает исполнение, в случае возвращения не нулевого результата исполнения любой команды в последовательности, кроме последней
- "-u" - все не заданные переменные считаются ошибкой
- "-x" - включает возможности вывода всех исполняемых команд в терминал
- "-o pipefail" - переопределяет вывод результата исполнения команд, т.о. что если одна из команд в последовательности падает в ошибку, то возвращается ее код ошибки, а не результат последней команды в последовательности

По факту такой режим использовать имеет смысл в ходе написания и отладки сценариев, т.к. он позволяет более подробно увидеть место, где возникает ошибка.

## Используя -o stat для ps, определите, какой наиболее часто встречающийся статус у процессов в системе. В man ps ознакомьтесь (/PROCESS STATE CODES) что значат дополнительные к основной заглавной буквы статуса процессов. Его можно не учитывать при расчете (считать S, Ss или Ssl равнозначными).

      vagrant@vagrant:~$ ps -o stat
      STAT
      Ss
      R+

По выводу получается Ss - обычный спящий процесс, который может быть прерван, ожидает какого-то события, s - в мане написано "is a session leader", т.е. лидер сеанса...

R - исполняется или ожидает исполнения, + - исполняется в фоне