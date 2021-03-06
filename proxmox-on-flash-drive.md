# Установка Proxmox на SD Card

Открытая система виртуализации [Proxmox](htttps://proxmox.com) не предоставляет удобного варианта установки на Flash Card без трагических последствий для самой Flash Card, как, например, это делает VMware ESXi. В статье я попробовал описать моменты установки Proxmox на Flash Card, призванные минимизировать расходование ресурсов флеш-карты во время дальнейшей эксплуатации.

Для определенности допустим, что необходимо запустить Proxmox на сервере без жестких дисков, внутри которого установлена карта SD объемом 16 Гб. Так же, в наличии имеется хранилище, предназначенное для работы с настраиваемым виртуальным хостом.

## Установка Proxmox

Процесс установки с Live CD на карту SD ничем не отличается от установки на жесткий диск (во время установки следует выбрать файловую систему EXT4).

## Настройка системы

После окончания установки и перезагрузки необходимо внести изменения в файл **/etc/fstab**:
* Для уменьшения числа операций записи при обращении к файлам необходимо добавить параметр монтирования **noatime** для всех файловых систем, расположенных на Flash Card.
* Раздел **swap** следует отключить полностью. Proxmox весьма часто пишет в swap и это может привести к очень быстрому выходу карты памяти из строя.

В результате должно получиться примерно так:
```
# <file system> <mount point> <type> <options> <dump> <pass>
/dev/pve/root / ext4 errors=remount-ro,noatime 0 1
proc /proc proc defaults 0 0
```

Далее, продолжая уменьшать негативное воздействие на Flash Card операций записи, необходимо несколько директорий перенастроить на работу в tmpfs. Для этого в файл /etc/fstab нужно добавить строки:
```
tmpfs /tmp tmpfs defaults,nodev,nosuid 0 0
tmpfs /var/tmp tmpfs defaults,nodev,nosuid 0 0
tmpfs /var/lib/rrdcached tmpfs defaults,nodev,nosuid 0 0
tmpfs /var/lib/vz tmpfs defaults,nodev,nosuid 0 0
tmpfs /var/lib/vzctl tmpfs defaults,nodev,nosuid 0 0
tmpfs /var/lib/vzquota tmpfs defaults,nodev,nosuid 0 0
```

Размер tmpfs можно не указывать, так как объем файлов в этих директориях невелик. _Главное помнить, что нельзя локально создавать виртуальные серверы, у нас же нет hdd!_.

В директории **/var/log** файлы так же являются часто перезаписываемыми, но размещать эту директорию в tmpfs не совсем разумно, так как объем этой директории может быть большим. Вариантов решения может быть несколько, например:
- Разместить /var/log на сетевой файловой системе (наример, NFS).
- Настроить удаленное логирование на выделенный лог-сервер.

Перенести /var/log на сетевую файловую систему NFS можно следующим образом:
* создать на сетевом хранилище директорию и примонтировать ее на сервер Proxmox во временную директорию /mnt/tmp с помощью NFS:
  ```
  mount -t nfs -o vers=3,noatime,soft,intr X.X.X.X:/path/to/log/dir /mnt/tmp
  ```
* остановить rsyslog:
  ```
  service rsyslog stop
  ```
* перенести все содержимое текущей директории /var/log в примонтированную:
  ```
  mv /var/log/* /mnt/tmp
  ```
* отмонтировать сетевую директорию из /mnt/tmp и примонтировать в /var/log:
  ```
  umount /mnt/tmp
  mount -t nfs -o vers=3,noatime,soft,intr X.X.X.X:/path/to/log/dir /var/log
  ```
* запустить rsyslog:
  ```
  service rsyslog start
  ```

Чтобы директория /var/log монтировалась автоматически при рестарте сервера, нужно добавить строку в файл /etc/fstab:
```
X.X.X.X:/path/to/log/dir /var/log nfs vers=3,noatime,soft,intr 0 0
```

Осталось выполнить перезагрузку сервера, после которой можно начинать эксплуатировать сервер Proxmox.

