В наше время, только ленивый не защищает свои личные данные и не печется о своей приватности. Но мало кто использует для этого полнодисковое шифрование, из-за чего вся защита легко снимается при наличии физического доступа к компьютеру. Сегодня мы разберемся, как можно зашифровать весь жесткий диск, включая загрузчик, оставив себе возможность быстрого уничтожения данных без физического доступа к ним. А также рассмотрим, как защитить и анонимизировать сетевой трафик. Про Tor слышали уже практически все, многие даже использовали его как прокси для отдельных приложений. Мы же покажем, как эффективно пользоваться им для полной анонимизации целой операционной системы.




# Шифрование




## Требования к системе


Мы считаем, что хорошая криптосистема должна шифровать абсолютно всё содержимое жесткого диска так, чтобы его содержимое нельзя было отличить от случайных данных. Загрузчик системы должен состоять из двух частей и находиться на флешке или другом сьёмном носителе. Первая, незащищенная часть, по паролю расшифровывает вторую, в которой находится ядро системы и ключи от жесткого диска. Система должна допускать быструю смену ключей и паролей без потери данных, а работа с разделами должна быть такой же, как и обычно.


## Использование


Наша схема работает следующим образом: при включении компьютера мы вставляем в него флешку, она запускает загрузчик, вводится пароль, а после загрузки ОС флешка вынимается и работа с компьютером идёт как обычно. В то время как обычные средства шифрования защищают только домашний раздел или оставляют загрузчик на жестком диске эта схема делает выключенный компьютер практически неуязвимым к снятию информацию и протрояниванию компонентов системы, а для уничтожения всех данных на нём нужно просто сломать флешку. В то время, когда компьютер включен, ключи от диска находятся в оперативной памяти, поэтому для надёжной защиты от cold-boot атак необходимо использовать пароль на BIOS/UEFI и входить в режим гибернации, если нет возможности контролировать доступ к компьютеру.




## Подготовка диска




Нам понадобится:




1. Live-DVD с любым дистрибутивом Linux (команды даны для Debian/Ubuntu)
2. Любая свободная флешка, размером более 128 Мб
3. Прямые руки




Если ты хочешь защитить уже существующую систему, а не ставить новую, то необходимо сделать её полный бэкап. Для начала нам нужно загрузиться с Live-DVD и определиться с носителями, на которые мы будем ставить систему. В нашем примере мы работаем с дистрибутивом Debian, а наши носители: `/dev/sda` - жесткий диск, `/dev/sdb` - флеш-карта.
Теперь нам необходимо получить консоль с правами root и установить необходимые программы (cryptsetup – полнодисковое шифрование, grub2 – загрузчик)


        apt-get install cryptsetup grub2


Если на жестком диске была какая-то незашифрованная информация, то его необходимо полностью перезаписать случайными данными, для этого определим его размер в байтах


        fdisk -l /dev/sda


И затираем данные на диске случайными данными


        openssl rand $размер_в_байтах | dd of=/dev/sda bs=512K


Теперь нам необходимо сгенерировать ключ и положить его в `/root/key`


        dd if=/dev/random of=/root/key bs=1 count=32


Форматируем жесткий диск


        cryptsetup luksFormat /dev/sda key


В начале диска, зашифрованного LUKS, есть заголовок, в котором перечислены методы шифрования и зашифрованные ключи. Чтобы содержимое диска выглядело как случайные данные, мы забэкапим и затрём этот заголовок.


        cryptsetup luksHeaderBackup /dev/sda --header-backup-file /root/header.luks
        dd if=/dev/urandom of=/dev/sda bs=1 count=1052672


Открываем наш зашифрованный диск


        cryptsetup -d=key --header=header.luks luksOpen /dev/sda rootfs


C `/dev/mapper/rootfs` далее можно работать как с обычным жестким диском,
при необходимости можно использовать RAID, LVM, можно также создать swap-раздел.
Форматируем диск, устаналиваем систему или копируем защищаемую


        mkdir /mnt/root
        mkfs.ext4 /dev/mapper/rootfs
        mount /dev/mapper/rootfs /mnt/root


## Установка системы


В данном примере мы устанавливаем Debian Jessie с помощью debootstrap, утилиты для развёртывания базовой debian-based системы в папке другой системы. Копировать уже существующую систему можно rsync-ом.


        apt-get install debootstrap
        debootstrap --arch amd64 jessie /mnt/root http://http.debian.net/debian


Теперь chroot-имся в неё, ставим необходимый софт и настраиваем initramfs.


        chroot /mnt/root
        apt-get install vim locales linux-image-3.16.0-4-amd64(или более новую версию) cryptsetup
        cp /usr/share/initramfs-tools/hooks/cryptroot /etc/initramfs-tools/hooks
        CRYPTSETUP=y update-initramfs -k all -u


Устаналиваем пароль на систему (если она новая)
    
        passwd


Правим информацию о файловых системах в 'fstab', т.к. теперь корневая ФС системы при загрузке будет находиться в `/dev/mapper/rootfs`


        vim /etc/fstab
        /dev/mapper/rootfs / ext4 errors=remount-ro 0 1


Теперь выходим из chroot и делаем копию initrd


        cp /mnt/rootfs/boot/initrd-(текущая версия) /tmp/initrd
        mkdir /tmp/newinitrd
        cd /tmp/newinitrd


Распаковываем его


        zcat ../initrd | cpio -idv


Подправляем скрипты, запускаемые при загрузке системы до монтирования основных файловых систем[h]


В файле ./scripts/local-top/ORDER необходимо заменить cryptroot на crypto
Содержимое файла ./scripts/local-top/crypto нужно заменить на скрипт, открывающий наш полностью зашифрованный диск


        prereqs()
        {
            echo "$PREREQ"
        }
        case $1 in
        prereqs)
        prereqs
        exit 0
        ;;
        esac
        modprobe -b dm_crypt
        modprobe -b aes_generic
        modprobe -b sha256
        /sbin/cryptsetup -d=/etc/key --header=/etc/header.luks \
        luksOpen /dev/disk/by-id/$DISK_ID$ rootfs


`$DISK_ID` можно узнать, используя `ls -l /dev/disk/by-id/ | grep sda`




Делаем скрипт исполняемым:


        chmod +x ./scripts/local-top/crypto


Кладём в initrd ключ и заголовок тома


        cp /root/key /mnt/rootfs/initrd/etc/key
        cp /root/header.luks /mnt/rootfs/initrd/header.luks


Собираем получившийся initrd и кладём его в папку /root


        find . | cpio -H newc -o > ../initrd
        cd ..
        gzip initrd
        mv initrd.gz /root/initrd


Начинаем размечать флешку


        parted
        (parted) select /dev/sdb
        (parted) mklabel msdos
        (parted) mkpart primary 5 100 # Раздел с любой файловой системой, например fat32
        (parted) mkpart primary 100 200 # раздел с initrd и ядром, защищен luks
        (parted) mkpart primary 200 250 #раздел с grub


Теперь работаем с флешкой, шифруем раздел и устаналиваем на него пароль


        cryptsetup luksFormat /dev/sdb2


Осталось настроить загрузчик


        vim /etc/default/grub


добавляем в конец файла строчку


        GRUB_ENABLE_CRYPTODISK=1


Форматируем раздел под загрузчик


        mkfs.ext2 /dev/sdb3


Монтируем его


        mkdir /mnt/grub
        mount /dev/sdb3 /mnt/grub


Устаналиваем GRUB на флеш-карту, --target следует выбрать в зависимости от параметров компьютера, i386-pc наиболее универсальный вариант


        grub-install --target=i386-pc --root-directory /mnt/grub/ /dev/sdb


Теперь создадим файл конфигурации /mnt/grub/grub.cfg


        set timeout=5
        set default=0
        menuentry "Linux" {
            insmod cryptodisk
            insmod luks
            cryptomount hd0,msdos2
            set root=(crypto0)
            linux /vmlinuz root=/dev/mapper/rootfs ro quiet
            initrd /initrd
        }
   
  В нём мы пишем, что нужно создать одну строку меню "Linux", при запуске которой после ввода пароля открывается зашифрованный раздел флешки с ядром системы и ключами от жесткого диска.


Открываем защищенный раздел на флешке


        cryptsetup luksOpen /dev/sdb2 cryptboot


Форматируем его


        mkfs.ext2 /dev/mapper/cryptboot


Монтируем


        mkdir /mnt/cryptboot
        mount /dev/mapper/cryptboot /mnt/cryptboot


Копируем в него ядро и initrd


        cp /root/initrd /mnt/cryptboot/initrd
        cp /mnt/rootfs/boot/vmlinuz-(вставьте сюда свою версию) /mnt/cryptboot/vmlinuz
        sync


Всё готово, размонтируем диски и перезагружаемся


        umount /mnt/*
        cryptsetup luksClose /dev/mapper/cryptboot
        cryptsetup luksClose /dev/mapper/rootfs
        sync
        shutdown -r now