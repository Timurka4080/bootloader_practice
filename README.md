
# Инструкции
● Для проверки надо скачать репозиторий 

	cd ~
    git clone git@github.com:Timurka4080/bootloader_practice.git
	
● После этого зайдите в директорию bootloader_practice и запускаем виртуальную машину с помощью vagrant

	cd ~/bootloader_practice/
	vagrant up 
 
## 1. Сбрасываем пароль _root_
● В конце строки начинающейсā с __linux16__ добавлāем __rd.break__ и нажимаем _сtrl-x_ для загрузки в систему

● Попадаем в _emergency mode_. Наша корневая файловая система смонтирована (опять же в режиме **Read-Only**, но мы не в ней). Далее монтируем с параметром **rw** попадаем в нее и меняем пароль администратора:

	mount -o remount,rw /sysroot
	chroot /sysroot
	passwd root
	touch /.autorelabel

● После чего можно перезагружаться и заходить в систему с новым паролем.	

## 2. Измение имени _VG_  в LVM
● Переименование  VG делается с помощю команды __vgrename__

    vgrename VolGroup00 OtusRoot

● Далее правим /etc/fstab, /etc/default/grub, /boot/grub2/grub.cfg. Везде заменāем старое название на новое.   

Пересоздаем __initrd image__, чтобы он знал новое название _Volume Group_ командой __mkinitrd__

    mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)

● После чего можем перезагружаться и если все сделано правильно, успешно грузимся с новым именем Volume Group и проверяем:

    [root@lvm ~]# vgs
     VG #PV #LV #SN Attr VSize VFree
     OtusRoot 1 2 0 wz--n- <38.97g 0

## 3. Добавляем модуль в _initrd_
● Скрипты модулей хранятся в каталоге /usr/lib/dracut/modules.d/. Для того чтобы добавить свой модуль создаем там папку с именем __01test__:

    mkdir /usr/lib/dracut/modules.d/01test

● В нее поместим два скрипта:

    cd /usr/lib/dracut/modules.d/01test
    cat >module-setup.sh<<EOF

    #------------------------------------------------
    #!/bin/bash
 
    check() {
        return 0 
    } 
    depends() {
        return 0 
    } 
    install() {
        inst_hook cleanup 00 "${moddir}/test.sh" 
    } 
    #------------------------------------------------
    EOF

    cat >test.sh<<EOF
    #------------------------------------------------
    #!/bin/bash 
    exec 0<>/dev/console 1<>/dev/console 2<>/dev/console 
    cat <<'msgend' 
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
    echo " continuing....
    #------------------------------------------------
    EOF

● Далее даем нашим скриптам права на исполнине

    chmod +x module-setup.sh
    chmod +x test.sh

● Пересобираем образ __initrd__

    mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)

● Можно проверить/посмотреть какие модули загружены в образ:

    lsinitrd -m /boot/initramfs-$(uname -r).img | grep test

● После чего можно пойти двумя путями для проверки:

* Перезагрузиться и руками выключить опции __rghb__ и __quiet__ и увидеть вывод
* Либо отредактировать grub.cfg убрав эти опции

● В итоге при загрузке будет пауза на 10 секунд и вы увидите пингвина в выводетерминала
