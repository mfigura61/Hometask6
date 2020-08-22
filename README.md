# Hometask6
Загрузка Linux

# Домашнее задание #
Работа с загрузчиком

Попасть в систему без пароля несколькими способами

Установить систему с LVM, после чего переименовать VG

Добавить модуль в initrd

(*). Сконфигурировать систему без отдельного раздела с /boot, а только с LVM Репозиторий с пропатченым grub: https://yum.rumyantsev.com/centos/7/x86_64/ PV необходимо инициализировать с параметром --bootloaderareasize 1m Критерии оценки: Описать действия, описать разницу между методами получения шелла в процессе загрузки. Где получится - используем script, где не получается - словами или копипастой описываем действия.

Установить систему с LVM

0. Установим систему Centos с minimal образа  на VM в VirtualBox с именем CentOS. На нем и будем экспериментировать.

## Попасть в систему без пароля несколькими способами ##

1.  Способ **init=/bin/sh**  (См Скрин1)

Заходим в загрузчик до старта системы (перед тем, как начнется загрузка нужно успеть нажать на клашиву e - как раз когда появляется меню выбора сценариев загрузки)В конце строки, начинающейся с linux16 добавляем init=/bin/sh и нажимаем сtrl-x для загрузки в систему.Попадаем в shell без пароля.  

Приветствие: sh-4.2#
Однако, при этом рутовая файловая система при этом монтируется в режиме Read-Only. Чтобы перемонтировать ее в режим Read-Write воспользуемся командой:
sh-4.2# mount -o remount,rw /

В результате увидим:

sh-4.2# mount | grep root

/dev/mapper/centos-root on /type xfs (rw,realtime,attr2,inode64,noquota)  

2.  Способ **rd.break**  (См Скрин2)

В конце строки начинающейся с linux16 добавляем rd.break и нажимаем сtrl-x для загрузки в систему.
Попадаем в emergency mode. Наша корневая файловая система смонтирована опять же в режиме Read-Only, но мы не в ней.Выполняем команду перемонтирования корня для чтения и записи - mount -o remount,rw /sysroot, далее chroot /sysroot.  

Теперь мы можем поменять пароль, выполнив команду passwd или passwd root.

После смены пароля необходимо создать скрытый файл .autorelabel в /, выполнив touch /.autorelabel, этот файл нужен, для того чтобы выполнить relabel файлов в системе.
Делаем ребут.Загрузка произошла, теперь мы можем зайти под root, введя измененный пароль.  

3. Способ.  **rw init=/sysroot/bin/sh**   
В строке начинающейся с linux16 заменяем ro на rw init=/sysroot/bin/sh и нажимаем сtrl-x для загрузки в систему. От прошлого примера отличается только тем, что файловая система сразу смонтирована в режим Read-Write  

4. Установка с LVM. На системе с LVM переименовываем VG.
Посмотрим, что есть в системе сейчас:

[root@localhost ~]# vgs

VG #PV #LV #SN Attr VSize VFree

centos 1 2 0 wz--n- <7.00g 0

Обращаем внимание на строку с именем *centos*

Приступим к переименованию:

[root@localhost ~]# vgrename centos OtusRoot

Volume group "centos" successfully renamed to "OtusRoot"

Подравим конфигурационные файлы:

[root@localhost ~]# sed -i 's/centos/otusroot/g' /boot/grub2/grub.cfg && sed -i 's/centos/otusroot/g' /etc/fstab

Пересоздаем initrd image, чтобы он знал новое название Volume Group*

[root@localhost ~]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
много всего...
...

*** Creating image file done ***

*** Creating initramfs image file '/boot/initramfs-3.10.0-1127.el7.x86_64.img' done ***

Перезагружаем систему и проверяем успешную загрузку с новым именем:

[root@localhost ~]# vgs

VG #PV #LV #SN Attr VSize VFree

OtusRoot 1 2 0 wz--n- <7.00g 0

5. Добавить модуль в initrd

Скрипты модулей хранятся в каталоге /usr/lib/dracut/modules.d/. Для того чтобы добавить свой модуль создаем там папку с именем 01test:

[root@localhost ~]# mkdir /usr/lib/dracut/modules.d/01test

В нее поместим два скрипта:

1.module-setup.sh - который устанавливает модуль и вызывает скрипт test.sh

2.test.sh - собственно сам вызываемый скрипт, в нём у нас рисуется пингвинчик.

[root@localhost ~]# ls -a

. .. module-setup.sh test.sh  

В скрипт module-setup.sh вписываем:  

```#!/bin/bash

check() { # Функция, которая указывает что модуль должен быть включен по умолчанию
    return 0
}

depends() { # Выводит все зависимости от которых зависит наш модуль
    return 0
}

install() {
    inst_hook cleanup 00 "${moddir}/test.sh" # Запускает скрипт
}```  


В файле test.sh:
 
```#!/bin/bash

exec 0<>/dev/console 1<>/dev/console 2<>/dev/console
cat <<'msgend'
Hello! You are in dracut module!
 ___________________
< I'm dracut module >
 -------------------
   \
    \
        .--.
       |o_o |
       |:_/ |
      //   \ \
     (|     | )
    /'\_   _/`\
    \___)=(___/
msgend
sleep 10
echo " continuing...."   
```

Теперь пересоберем образ initrd и при загрузке видим наш рисунок (Скрин3)

> [root@localhost ~]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)  

> [root@localhost ~]# dracut -f -v

