# 🛡️ Arch Linux + LUKS2 (argon2id) + BTRFS (subvolumes, zstd) + systemd-boot. Installation Guide

> ⚠️ **Внимание**:  
> Данный гайд описывает современную установку **Arch Linux** в режиме **UEFI** с полным шифрованием диска (**LUKS2**) и файловой системой **BTRFS**. Установка в режиме Legacy (BIOS/MBR) **не рассматривается**.
> 
> В качестве имени блочного устройства будем использовать - **/dev/nvme0n1**
---

## 🧪 Предварительные проверки

```bash
ls /sys/firmware/efi          # UEFI-режим? (каталог должен существовать)
ping archlinux.org            # наличие сетевого подключения
```

Если нет подключения — подключитесь к Wi-Fi через `iwctl`:

```bash
iwctl device list
iwctl station <device> scan
iwctl station <device> get-networks
iwctl station <device> connect <SSID>
```

Обновление зеркал (Россия, топ-25 по скорости):

```bash
reflector --verbose --country 'Russia' -l 25 --sort rate --save /etc/pacman.d/mirrorlist
```

---

## 💾 Подготовка диска

### Схема разметки:
| № | Тип | Размер | Код | Назначение |
|---|-----|--------|-----|------------|
| 1 | EFI System Partition | 1 GiB | `EF00` | FAT32 |
| 2 | Linux filesystem | остаток | `8300` | LUKS2 → BTRFS |

```bash
sgdisk -Z -o /dev/nvme0n1 # очистка данных GPT
sgdisk -n 1::+1G -t 1:EF00 -n 2:: -t 2:8300 /dev/nvme0n1
sgdisk -p /dev/nvme0n1 # вывод разметки
```

> ❗ **Swap не рекомендуется**  
> Swap-раздел — угроза безопасности: может раскрыть содержимое RAM. При ≥16 ГБ ОЗУ он избыточен.

---

## 🔐 Шифрование LUKS2

```bash
cryptsetup luksFormat /dev/nvme0n1p2 # создание контейнера
cryptsetup open /dev/nvme0n1p2 cryptroot # открытие контейнера
```


## 🌳 Файловая система (BTRFS)

```bash
# EFI
mkfs.fat -F32 /dev/nvme0n1p1

# Корневой том
mkfs.btrfs /dev/mapper/cryptroot

# Монтируем временно, создаём подтома
mount /dev/mapper/cryptroot /mnt

btrfs subvolume create /mnt/@            # системные файлы, поддерживает снапшоты
btrfs subvolume create /mnt/@home        # пользовательские данные, не включаются в снапшоты корня
btrfs subvolume create /mnt/@var_cache   # кеши пакетов, временные данные приложений (не нужны в снапшотах)
btrfs subvolume create /mnt/@var_log     № логи постоянно меняются. Исключаем из снимков, имеем новые логи логи для диагностики при откате системы
btrfs subvolume create /mnt/@var_tmp     # временные файлы (не нужны в снапшотах)
btrfs subvolume create /mnt/@snapshots   # ручные/автоматические снапшоты (для snapper)

umount /mnt
```

## 📦 5. Монтирование

```bash
# Корень
mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvol=@ \
  /dev/mapper/cryptroot /mnt

# Подтома
mkdir -p /mnt/{home,var/log,var/tmp,boot,.snapshots}
mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvol=@home          /dev/mapper/cryptroot /mnt/home
mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvol=@var_log       /dev/mapper/cryptroot /mnt/var/log
mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvol=@var_tmp       /dev/mapper/cryptroot /mnt/var/tmp
mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvol=@snapshots     /dev/mapper/cryptroot /mnt/.snapshots
mount /dev/nvme0n1p1 /mnt/boot
```

### 📌 Параметры монтирования:
| Параметр | Эффект |
|----------|--------|
| `noatime` | Отключает обновление `atime`, снижает I/O и износ SSD |
| `compress=zstd` | Сжатие Zstandard (уровень 3 по умолчанию) |
| `space_cache=v2` | Современный кэш свободного места — быстрее аллокации |
| `commit=120` | sync каждые 120 сек → меньший IOPS, но риск потери данных при аварийном отключении |
| `subvol=@…` | Выбор подавтоматома для точки монтирования |

> 💡 **BTRFS-подавтом и снапшоты**  
> Подавтома позволяют делать снапшоты корня (`@`) без затрагивания `/home` (`@home`). Для автоматизации — рассмотрите [snapper](http://snapper.io).

---

## 📦 Установка базовой системы

```bash
pacstrap -K /mnt base linux-zen amd-ucode neovim # базовые инструменты, ядро linux-zen, микрокод для AMD и современный редактор nvim
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

### Настройка окружения

```bash
# Обновление
pacman -Syu

# Часовой пояс
ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime
hwclock --systohc

# Локали (ru_RU.UTF-8, en_US.UTF-8)
sed -i 's/^#\?\(ru_RU\.UTF-8\|en_US\.UTF-8\)/\1/' /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "KEYMAP=en" > /etc/vconsole.conf

# Хост
HOSTNAME="arch-pc"
echo "$HOSTNAME" > /etc/hostname
cat > /etc/hosts <<EOF
127.0.0.1	localhost
::1		localhost
127.0.1.1	$HOSTNAME.localdomain	$HOSTNAME
EOF

# Пользователь
pacman -S --needed sudo
EDITOR=nvim visudo  # раскомментировать: %wheel ALL=(ALL:ALL) ALL
passwd
useradd -mG wheel sol
passwd sol
```

### Сеть

```bash
pacman -S networkmanager
systemctl enable NetworkManager
systemctl mask NetworkManager-wait-online

# Отключить connectivity check (пинг archlinux.org)
mkdir -p /etc/NetworkManager/conf.d
echo -e "[connectivity]\nenabled=false" > /etc/NetworkManager/conf.d/10-disable-connectivity-check.conf
```
> 💡 **Отключение NetworkManager-wait-online**  
> Ускоряет загрузку системы, т.к. система не ждёт от NetworkManager сообщения о наличии активного сетевого подключения. Нужно включить, если в момент запуска есть зависящие от сети службы (NFS, SMB и др.)

---

## 🔧 Initramfs и загрузчик

### `/etc/mkinitcpio.conf`
Замените `HOOKS` на:

```conf
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)
```

Сборка:

```bash
mkinitcpio -P
```

### `systemd-boot`

```bash
bootctl install

# /boot/loader/loader.conf
cat > /boot/loader/loader.conf <<'EOF'
default arch.conf
timeout 3
console-mode max
editor no
EOF

# /boot/loader/entries/arch.conf
UUID=$(blkid -s UUID -o value /dev/nvme0n1p2)
cat > /boot/loader/entries/arch.conf <<EOF
title Arch Linux (zen)
initrd /amd-ucode.img
initrd /initramfs-linux-zen.img
linux /vmlinuz-linux-zen
options rw rd.luks.name=$UUID=cryptroot root=/dev/mapper/cryptroot rootflags=subvol=@ loglevel=2 quiet
EOF
```

Проверка:
```bash
bootctl status
```

---

## 🚀 Завершение

```bash
exit                      # выход из chroot
umount -R /mnt
cryptsetup close cryptroot
reboot
```

---

## ❓ FAQ & Maintenance

### 💡 Q: zstd vs lzo?
**`zstd`**. Лучшая скорость/степень сжатия. Использует уровень 3 (`zstd:3`) по умолчанию

### 🔧 Q: Установка yay (AUR)
```bash
sudo pacman -Syu
sudo pacman -S --needed base-devel git
git clone https://aur.archlinux.org/yay.git
cd yay && makepkg -si
```

### 🧹 Очистка пакетного кеша

| Команда | Эффект |
|---------|--------|
| `sudo pacman -Sc` | Устаревшие версии пакетов |
| `sudo pacman -Scc` | Полная очистка (/var/cache/pacman/pkg) |
| `sudo pacman -Rns $(pacman -Qdtq)` | Орфаны (неиспользуемые зависимости) |
| `yay -Sc` / `yay -Scc` | Аналогично для AUR |

### ⌨️ Курсор «прыгает» при вводе пароля LUKS?
→ **Нормальное поведение**: systemd-boot не отображает символы и не двигает курсор.

→ **Альтернатива** — использование патченного `grub-improved-luks2-git` из AUR, т.к. GRUB не поддерживает `argon2id` (стандарт в LUKS2), но это выходит за рамки данного гайда.

### 📊 Мониторинг BTRFS

```bash
btrfs filesystem usage /      # использование пространства
btrfs filesystem df /         # распределение по типам данных
btrfs filesystem balance start /  # ← осторожно: I/O-нагрузка! Читай: [балансировка](https://wiki.archlinux.org/title/Btrfs_(Русский)#Балансировка)
```

> **Не забудьте проверить `journalctl -b` после первой загрузки** — ищите ошибки и недостающие модули.

---
