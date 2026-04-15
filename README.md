# Arch Linux на USB флешке (Archlinux on USB Stick)
Установка полноценного Archlinux и настройка системы для работы с USB-носителя с целью минимизации износа памяти. Оптимизировано для нетбука Asus L203m с малым объемом RAM и неисправным внутренним диском.

## Особенности сборки
- Минимизация износа флешки: логи и временные файлы в RAM, отключение лишних записей на диск.
- Производительность: использование zRAM и легковесного окружения Hyprland.
- Портативность: установка загрузчика в режиме --removable позволяет запускать систему на разных пк.

## Железо
- Нетбук Asus L203m Celeron N4000, 4GB RAM, Отвал SSD
- Флешка Verbatim Nano 32GB USB 3.2
- Установочная флешка с Archlinux сделанная любым удобным способом

## Загрузочная флешка
ВНИМАНИЕ: Обязательно проверить имя диска. Все файлы будут удалены!
```bash
# скачиваем образ Archlinux
# Найти свою флешку
lsblk
# Запись образа (замени sdx на свою)
sudo dd if=archlinux-x86_64.iso of=/dev/sdx bs=4M status=progress oflag=sync
``` 

## Подключение к Wifi через iwctl
``` bash
iwctl
# Найти WiFi адаптера
device -list
# Скан сети (замени wlan0 на свой)
station wlan0 scan
# Список найденных сетей
station wlan0 get-networks
# Подключение (замени NAME на имя своей сети)
station wlan0 connect NAME
# Вводим пароль и выходим
exit
# Проверка соединения
ping -c 5 archlinux.org
```

## Разметка диска 
``` bash
# Создаем два раздела EFI, ROOT (sdx замени на свой)
gdisk /dev/sdx
# если предложит изменить MBR на разметку на GPT подтверждаем для UEFI 
o
# Создаем раздел EFI (Partition number:1; last sector:+512M; Hex code:ef00)
n
# Создаем раздела ROOT (Partition number:2; last sector:все оставшееся место; Hex code:8300)
n
# Запись таблицы на диск
w
```

## Форматирование разделов
``` bash
# EFI раздел (sdx замени на свой)
mkfs.fat -F32 /dev/sdx1
# BOOT раздел (sdx замени на свой)
mkfs.ext4 /dev/sdx2
```

## Монтирование разделов
``` bash
# sdx замени на свой
mount /dev/sdx2 /mnt
mkdir /mnt/boot
mount /dev/sdx1 /mnt/boot
```

## Установка базовой системы
``` bash
# Качаем пакеты
pacstrap /mnt base base-devel linux linux-firmware intel-ucode networkmanager git nvim  
# Генерируем fstab
genfstab -U /mnt >> /mnt/etc/fstab
# Вход в систему 
arch-chroot /mnt
# Часовой пояс
ln -sf /usr/share/zoneinfo/Europe/Amsterdam /etc/localtime 
hwclock --systohc
# Локаль
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
# Имя машины
echo "arch-nano" > /etc/hostname
# Исправляем ошибку с vconsole
echo "KEYMAP=us" > /etc/vconsole.conf
```

## Пароль root и пользователи
``` bash
# Пароль root 
passwd
# создаем пользователя (замени NAME на свой)
useradd -m -G wheel -s /bin/bash NAME 
passwd NAME
# Даем sudo доступ (раскомментировать: %wheel ALL=(ALL:ALL) ALL)
EDITOR=nvim visudo
```

## Установка загрузчика 
``` bash
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB --removable
grub-mkconfig -o /boot/grub/grub.cfg
# Исправление ошибки initramfs
mkinitcpio -P
```

## Включаем NetworkManager и настройка tmpfs для логов
``` bash
# Включаем NetworkManager
systemctl enable NetworkManager
# Настройка логов в RAM
echo "Storage=volatile" >> /etc/systemd/journald.conf
echo "RuntimeMaxUse=64M" >> /etc/systemd/journald.conf
```

## Настройка zram в RAM
``` bash 
pacman -S zram-generator
cat > /etc/systemd/zram-generator.conf << EOF
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
EOF
```

## Выход и перезагрузки
``` bash
exit
umount -R /mnt
reboot
# После перезагрузки вытягиваем загрузочную флешку и заходим под пользователем. Подключаем WiFi
nmtui
```

## Hyprland Драйвера и все что нужно
``` bash
# Установка необходимых пакетов 
sudo pacman -S hyprland kitty mako firefox openssh pipewire pipewire-pulse wireplumber xdg-desktop-portal-hyprland qt5-wayland qt6-wayland waybar wofi
# Создаем конфиг Hyprland
mkdir -p ~/.config/hypr
nvim ~/.config/hypr/hyprland.conf
# Минимальный конфиг
monitor=,preferred,auto,1
input {
    kb.layout = us
    touchpad {
      natural_scroll = true
    }
}
general {
    gaps_in = 5
    gaps_out = 10
    border_size = 2
}
misc {
  disable_hyprland_logo = true
}
bind = $SUPER, Return, exec, kitty
bind = SUPER, Q, killactive
bind = SUPER SHIFT, E, exit
bind = SUPER, B, exec, firefox
# Запускаем!
Hyprland
# Автозапуск Hyprland
nvim ~/.bash_profile
if [ -z "$WAYLAND_DISPLAY" ] && [ "$XDG_VTNR" = "1" ]; then
  exec start-hyprland
fi
# Включаем звук
systemctl --user enable --now pipewire pipewire-pulse wireplumber
```

## Сокращаем журналирование ext4
``` bash
sudo nvim /etc/fstab
# Найти и добавить к опциям
UUID=xxx / ext4 rw,noatime,commit=60 0 1
# Добавить в конец файла 
tmpfs /tmp tmpfs defaults,noatime,mode=1777 0 0
tmpfs /var/tmp tmpfs defaults,noatime,mode=1777 0 0
```

## Firefox + psd 
``` code
# В строке браузера about:config изменяем парраметры
browser.cache.disk.enable = false
browser.sessionstore.interval = 15000000
browser.cache.disk.capacity = 0
``` 
``` bash
# Перенос кеш браузера в RAM
rm -rf ~/.cache/mozilla
ln -s /tmp ~/.cache/mozilla
# Установка PSD правим конфиг и запускаем 
sudo pacman -S profile-sync-daemon
nvim ~/.config/psd/psd.conf
systemctl --user enable psd
systemctl --user start psd
psd -p
```
