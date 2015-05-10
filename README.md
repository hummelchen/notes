```bash
# /dev/sda - жесткий диск, /dev/sdb - флеш-карта
# Грузимся в live-dvd, получаем консоль с правами root, ставим необходимые программы
apt-get gparted debootstrap cryptsetup grub2
# Определяем количество байт, которыми нужно затереть информацию на диске
fdisk -l /dev/sda
# Затираем данные на диске случайными данными
openssl rand 8589934592 | dd of=/dev/sda bs=512K
# Генерируем ключ и форматируем жесткий диск
dd if=/dev/random of=/root/key bs=1 count=32
cryptsetup luksFormat /dev/sda key
cryptsetup luksHeaderBackup /dev/sda --header-backup-file /root/header.luks
cryptsetup -d=key luksOpen /dev/sda rootfs
# C /dev/mapper/rootfs далее можно работать как с обычным жестким диском 
# при необходимости можно использовать raid, lvm, рекомендуется создать swap-раздел
# Форматируем диск и устаналиваем систему
mkfs.ext4 /dev/mapper/rootfs
mount /dev/mapper/rootfs /mnt/root
# в нашем примере устанавливаемая ОС - Debian Jessie, архитектура - amd64
debootstrap --arch amd64 jessie /mnt/root http://http.debian.net/debian 
# Теперь chroot-имся в неё, ставим необходимый софт и настраиваем initramfs
chroot /mnt/
apt-get install vim locales linux-image-3.16.0-4-amd64 cryptsetup
cp /usr/share/initramfs-tools/hooks/cryptroot /etc/initramfs-tools/hooks
CRYPTSETUP=y update-initramfs -k all -u
# устаналиваем пароль на систему
passwd 
#правим fstab
vim /etc/fstab #
/dev/mapper/rootfs /               ext4    errors=remount-ro 0       1
# выходим из chroot и разбираем initrd
cp /mnt/rootfs/boot/initrd /tmp/initrd
mkdir /tmp/newinitrd
cd /tmp/newinitrd
zcat ../initrd | cpio -idv
vim ./scripts/local-top/ORDER # Необходимо заменить cryptroot на crypto
vim ./scripts/local-top/crypto
#####################################
prereqs()
{
echo "$PREREQ"
}
case $1 in
# get pre-requisites
prereqs)
prereqs
exit 0
;;
esac
modprobe -b dm_crypt
modprobe -b aes_generic
modprobe -b sha256
/sbin/cryptsetup -d=/etc/key --header=/etc/header.luks luksOpen /dev/disk/by-id/$DISK_ID$ rootfs
# $DISK ID можно узнать, используя ls -l /dev/disk/by-id/ | grep sda$
####################################
chmod +x ./scripts/local-top/crypto
# кладём в initrd ключ и header
cp /root/key /mnt/rootfs/initrd/etc/key
cp /root/header.luks /mnt/rootfs/initrd/header.luks 
# Собираем initrd
find . | cpio -H newc -o > ../initrd
cd ..
gzip initrd
mv initrd.gz /root/initrd
parted 
(parted) select /dev/sdb
(parted) mklabel msdos # при необходимости можно использовать и GPT                                                   
(parted) mkpart primary 5 100  #Раздел с любой файловой системой, например fat32
(parted) mkpart primary 100 200 #раздел с initrd и ядром, защищен luks 
(parted) mkpart primary 200 250  #раздел с grub
# Теперь работаем с флешкой, форматируем раздел и устаналиваем на него пароль
cryptsetup luksFormat /dev/sdb2 
vim /etc/default/grub #добавляем в конец файла строчку GRUB_ENABLE_CRYPTODISK=1
mkfs.ext2 /dev/sdb3
mkdir /mnt/grub
mount /dev/sdb3 /mnt/grub
# устаналиваем GRUB на флеш-карту
grub-install --target=i386-pc --root-directory /mnt/grub/ /dev/sdb mv /mnt/grub/boot/grub /mnt/grub/grub
vim /mnt/grub/grub.cfg
# Создаём файл с следующим содержанием
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
# Открываем защищенный раздел на флешке
cryptsetup luksOpen /dev/sdb2 cryptboot
mkfs.ext2 /dev/mapper/cryptboot
mkdir /mnt/cryptboot
mount /dev/mapper/cryptboot /mnt/cryptboot
cp /root/initrd /mnt/cryptboot/initrd
cp /mnt/rootfs/boot/vmlinuz-3.16.0-4-amd64 /mnt/cryptboot/vmlinuz
sync
# Всё готово, размонтируем диски
umount /mnt/*
cryptsetup luksClose /dev/mapper/cryptboot
cryptsetup luksClose /dev/mapper/rootfs
#Затираем luksHeader, чтобы содержание диска ничем не отличалось от случайных данных
dd if=/dev/urandom of=/dev/sda bs=1 count=1052672
```
