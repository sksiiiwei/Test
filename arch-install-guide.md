# Установка Arch Linux + Hyprland вместо Windows — подробный план

Рекомендация по DE/WM: **Hyprland**. Это Wayland-компоузитор, сейчас самый популярный выбор для рисов (динамические тайлы, анимации, blur, гибкий конфиг), плюс активное коммьюнити и куча готовых дотфайлов на GitHub под него. Экосистема: Waybar (панель), Wofi/Rofi (лаунчер), Hyprpaper/Swww (обои), Kitty/Alacritty (терминал), SwayLock (лок-экран).

---

## 0. Бэкап данных

Раз Windows сносится полностью — сначала сохрани всё нужное на внешний носитель или в облако:

- Личные файлы (Документы, Рабочий стол, Загрузки) → внешний SSD/флешка или облако (Google Drive, Яндекс.Диск и т.д.)
- Экспортируй пароли из браузера / менеджера паролей
- Сохрани лицензионные ключи софта, если есть платный
- Если есть BitLocker — отключи шифрование диска перед манипуляциями (Панель управления → Шифрование диска BitLocker → Отключить)
- Проверь бэкап на другом устройстве, что файлы реально читаются — не ленись, это единственный шанс всё не потерять

---

## 1. Подготовка установочного носителя

**Скачай образ:**
- https://archlinux.org/download/ — выбери ближайшее зеркало, скачай `.iso`

**Проверь целостность (важно для ИБ-специалиста — делай это по привычке):**
```bash
# на Linux/WSL
sha256sum archlinux-x86_64.iso
# сравни с checksum на сайте архива
```
Можно также проверить GPG-подпись — на странице загрузки есть инструкция и публичный ключ.

**Запиши на флешку (от 4 ГБ):**

На Windows — **Rufus**:
1. Скачай Rufus (rufus.ie)
2. Выбери флешку, выбери скачанный ISO
3. Partition scheme: **GPT**, Target system: **UEFI**
4. Режим записи — **DD Image** (не ISO Image), когда Rufus спросит — это важно для Arch ISO
5. Start, дождись записи

---

## 2. Настройка BIOS/UEFI

Перезагрузись и зайди в BIOS (обычно `Del`, `F2` или `F10` при загрузке — зависит от материнки/ноутбука):

1. **Отключи Secure Boot** (Arch по умолчанию не поддерживает его без доп. настройки)
2. Убедись, что режим загрузки **UEFI**, не Legacy/CSM
3. В Windows перед этим шагом также стоило отключить **Fast Startup** (Панель управления → Электропитание → Действия кнопок питания → снять галочку «Быстрый запуск») — иначе диск может остаться в «грязном» состоянии
4. Выстави флешку первым приоритетом загрузки, или используй Boot Menu (обычно `F12`/`F11`) для разового выбора

---

## 3. Загрузка с флешки и подготовка

Загрузишься в live-окружение Arch (консоль). Дальше всё по шагам:

**Проверка, что загрузились в UEFI-режиме:**
```bash
ls /sys/firmware/efi/efivars
```
Если папка есть и не пустая — всё ок, UEFI.

**Настрой раскладку клавиатуры (опционально, для русской):**
```bash
loadkeys ru
```

**Подключение к интернету:**

Если Ethernet-кабель — обычно работает сразу. Проверь:
```bash
ping -c 3 archlinux.org
```

Если Wi-Fi:
```bash
iwctl
# внутри iwctl:
device list
station <device> scan
station <device> get-networks
station <device> connect "<SSID>"
# введи пароль, затем exit
```

**Синхронизация времени:**
```bash
timedatectl set-ntp true
```

---

## 4. Разметка диска (полное удаление Windows)

⚠️ Это необратимо удалит Windows и все данные на диске. Убедись, что бэкап сделан.

**Посмотри список дисков:**
```bash
lsblk
```
Найди свой диск (например `/dev/sda` или `/dev/nvme0n1` для NVMe SSD).

**Зайди в разметку (cfdisk — самый наглядный вариант):**
```bash
cfdisk /dev/nvme0n1
```

1. Выбери **gpt** как тип таблицы разделов (если было MBR от старой Windows — пересоздай)
2. Удали все существующие разделы (Delete на каждом)
3. Создай новые разделы (**всего два** — под своп отдельную партицию не делаем, сделаем файл подкачки чуть ниже, это гибче):
   - **EFI-раздел**: 512 МБ–1 ГБ, тип `EFI System`
   - **Корневой раздел**: всё оставшееся место, тип `Linux filesystem`
4. Write → введи `yes` → Quit

**Форматирование:**
```bash
mkfs.fat -F32 /dev/nvme0n1p1      # EFI
mkfs.ext4 /dev/nvme0n1p2           # корень
```

**Монтирование:**
```bash
mount /dev/nvme0n1p2 /mnt
mkdir -p /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

**Файл подкачки (swap) — делаем прямо сейчас, до установки базовой системы:**

Раньше своп чаще делали отдельным разделом, сейчас стандартная практика — файл подкачки: его проще менять в размере, не нужно заранее гадать со сколько ГБ резервировать под раздел.

Ориентир по размеру (замени `count=8192` ниже под себя):
- ОЗУ ≤ 8 ГБ → своп = размеру ОЗУ (если нужен hibernate — x2)
- ОЗУ 8–32 ГБ → 4–8 ГБ свопа достаточно как подушка безопасности
- ОЗУ > 32 ГБ → можно вообще 2–4 ГБ или пропустить этот шаг

```bash
dd if=/dev/zero of=/mnt/swapfile bs=1M count=8192 status=progress   # 8192 = 8 ГБ, поменяй под себя
chmod 600 /mnt/swapfile
mkswap /mnt/swapfile
swapon /mnt/swapfile
```
⚠️ Если размер меняешь на не кратный 1 МБ — не страшно, `count` просто в мегабайтах, `count=4096` = 4 ГБ и т.д.

Дальше `genfstab` (шаг 5) сам подхватит этот файл в fstab, потому что он уже активирован через `swapon` — отдельно ничего прописывать не нужно.

---

## 5. Установка базовой системы

**Перед этим — ускорь зеркала (опционально, но сильно экономит время):**
```bash
pacman -S reflector --noconfirm
reflector --country Russia --latest 10 --sort rate --save /etc/pacman.d/mirrorlist
```
(замени `Russia` на нужную страну/соседние, если скорость так себе)

```bash
pacstrap -K /mnt base linux linux-firmware base-devel vim networkmanager sudo git intel-ucode amd-ucode
```

Это займёт какое-то время — качает и ставит ядро, базовые утилиты, компилятор (`base-devel` нужен для AUR в будущем).

⚠️ **Важно, что легко упустить:**
- `sudo` — без него следующие шаги (`visudo`) просто не сработают, пакет не ставится по умолчанию ни в `base`, ни в `base-devel`
- `git` — тоже не входит в `base-devel`, а без него не склонируешь ни `yay`, ни дотфайлы дальше по гайду
- `intel-ucode` / `amd-ucode` — микрокод процессора с патчами безопасности (в т.ч. закрывающими аппаратные уязвимости уровня Spectre/Meltdown-класса). Поставь оба — лишний просто не подключится к загрузке не под тот CPU, не критично, но если хочешь чище — ставь только свой (`intel-ucode` для Intel, `amd-ucode` для AMD)

**Сгенерируй fstab:**
```bash
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab    # проверь, что там пути корректные
```

**Зайди в новую систему (chroot):**
```bash
arch-chroot /mnt
```

---

## 6. Настройка системы (внутри chroot)

**Часовой пояс:**
```bash
ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime   # замени на свой регион
hwclock --systohc
```

**Локали:**
```bash
vim /etc/locale.gen
# раскомментируй: en_US.UTF-8 UTF-8 и ru_RU.UTF-8 UTF-8
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

**Клавиатура (консоль):**
```bash
echo "KEYMAP=ru" > /etc/vconsole.conf
```

**Хостнейм:**
```bash
echo "arch-pc" > /etc/hostname
cat >> /etc/hosts << EOF
127.0.0.1   localhost
::1         localhost
127.0.1.1   arch-pc.localdomain arch-pc
EOF
```

**Пароль root:**
```bash
passwd
```

**Создай пользователя:**
```bash
useradd -m -G wheel -s /bin/bash yourusername
passwd yourusername
```

**Дай пользователю sudo:**
```bash
EDITOR=vim visudo
# раскомментируй строку: %wheel ALL=(ALL:ALL) ALL
```

---

## 7. Загрузчик (systemd-boot — проще GRUB, раз только Arch на диске)

```bash
bootctl install
```

Создай конфиг загрузчика:
```bash
cat > /boot/loader/loader.conf << EOF
default arch.conf
timeout 3
console-mode max
editor no
EOF
```

Узнай UUID корневого раздела:
```bash
blkid /dev/nvme0n1p2
```

Создай запись загрузки (добавь строку с микрокодом — `intel-ucode.img` для Intel или `amd-ucode.img` для AMD, она должна идти **до** `initramfs-linux.img`):
```bash
cat > /boot/loader/entries/arch.conf << EOF
title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /amd-ucode.img
initrd  /initramfs-linux.img
options root=UUID=<твой-UUID-сюда> rw
EOF
```
(если ставил только один из ucode-пакетов — оставь только соответствующую строку, лишняя строка на несуществующий файл сломает загрузку)

**Включи NetworkManager, чтобы сеть работала после перезагрузки:**
```bash
systemctl enable NetworkManager
```

---

## 8. Первая загрузка

```bash
exit
swapoff /mnt/swapfile
umount -R /mnt
reboot
```
⚠️ `swapoff` обязателен именно перед `umount` — раздел с активным свопом на нём не размонтируется («target is busy»), а `umount -R` сам своп не выключает.

Вытащи флешку при перезагрузке. Если всё сделано верно — загрузится Arch, залогинься под своим пользователем.

Проверь интернет:
```bash
ping -c 3 archlinux.org
```
Если Wi-Fi — используй `nmtui` для подключения через NetworkManager.

---

## 9. Пост-установка: графика и AUR

**Обнови систему сразу после установки:**
```bash
sudo pacman -Syu
```

**Драйвер видеокарты:**
```bash
# NVIDIA
sudo pacman -S nvidia-dkms nvidia-utils nvidia-settings linux-headers

# AMD
sudo pacman -S mesa vulkan-radeon

# Intel
sudo pacman -S mesa vulkan-intel
```

⚠️ **Если у тебя NVIDIA** — учти, что связка NVIDIA + Wayland (а значит и Hyprland) исторически проблемная. Помимо установки `nvidia-dkms` (не `nvidia` — тот пакет жёстко привязан к конкретной версии ядра и ломается при обновлениях ядра), нужно ещё два шага, без которых велик шанс чёрного экрана при запуске Hyprland:

1. Добавь nvidia-модули в initramfs — открой `/etc/mkinitcpio.conf`, найди строку `MODULES=(...)` и впиши туда:
```
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```
Затем пересобери initramfs:
```bash
sudo mkinitcpio -P
```

2. Добавь параметр ядра для раннего kernel modesetting — открой `/boot/loader/entries/arch.conf` и в строке `options` добавь `nvidia_drm.modeset=1`:
```
options root=UUID=<твой-UUID> rw nvidia_drm.modeset=1
```

3. И в конфиг окружения (`~/.config/hypr/hyprland.conf` или `/etc/environment`):
```
env = LIBVA_DRIVER_NAME,nvidia
env = __GLX_VENDOR_LIBRARY_NAME,nvidia
env = NVD_BACKEND,direct
```

Если после всего этого всё равно будут проблемы с рендером — это известный класс багов, гугли по связке «nvidia hyprland» + версия драйвера, там почти всегда есть свежий воркэраунд.

**Аудио (pipewire — современный стандарт):**
```bash
sudo pacman -S pipewire pipewire-pulse pipewire-alsa wireplumber
```

**Установи yay (AUR-хелпер) — понадобится почти сразу для рисов и security-тулинга:**
```bash
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

---

## 10. Hyprland и окружение

```bash
sudo pacman -S hyprland waybar wofi kitty dolphin \
  xdg-desktop-portal-hyprland qt5-wayland qt6-wayland \
  polkit-kde-agent ttf-jetbrains-mono-nerd grim slurp \
  swaylock swww
```

- `waybar` — панель сверху
- `wofi` — лаунчер приложений
- `kitty` — терминал (популярен для рисов, поддерживает картинки/прозрачность)
- `grim` + `slurp` — скриншоты
- `swww` — плавная смена обоев
- `ttf-jetbrains-mono-nerd` — шрифт с иконками, нужен почти во всех рис-конфигах

⚠️ **Раскладка клавиатуры в GUI** — `vconsole.conf` из шага 6 работает только в консоли (TTY), на Hyprland её нужно задать отдельно в конфиге. После первого запуска открой `~/.config/hypr/hyprland.conf` и добавь в секцию `input`:
```
input {
    kb_layout = us,ru
    kb_options = grp:alt_shift_toggle
}
```
(переключение раскладки — Alt+Shift; если ставишь готовые дотфайлы с GitHub, эта секция там обычно уже есть — просто проверь и подправь под себя)

Запуск Hyprland из TTY после логина: просто набери `Hyprland`. Для автозапуска при логине можно настроить display manager (например `sddm`):
```bash
sudo pacman -S sddm
sudo systemctl enable sddm
```

---

## 11. Про «поставить с GitHub» — дотфайлы и оформление

Да, это стандартный способ. Дотфайлы (конфиги) под Hyprland+Waybar+Wofi и т.д. выкладывают целыми репозиториями. Общий процесс:

```bash
git clone https://github.com/username/dotfiles-repo.git ~/dotfiles
cd ~/dotfiles
```

Дальше — смотри `README.md` в репозитории, у каждого своя структура. Часто есть install-скрипт:
```bash
./install.sh
```

Или конфиги нужно вручную закинуть в `~/.config/`:
```bash
cp -r ~/dotfiles/hypr ~/.config/
cp -r ~/dotfiles/waybar ~/.config/
```

Хороший инструмент для управления дотфайлами — **GNU Stow**, позволяет симлинками раскладывать конфиги из одной папки:
```bash
sudo pacman -S stow
cd ~/dotfiles
stow hypr waybar wofi
```

**Куда смотреть за готовыми рисами под Hyprland:**
- Поищи на GitHub по тегам `hyprland-dotfiles`, `hyprland-rice`
- Популярные готовые сборки: **ML4W dotfiles**, **JaKooLit's Hyprland-Dots**, **end-4/dots-hyprland** — у всех подробные README с инструкцией установки одной командой
- r/unixporn — там люди выкладывают скриншоты и ссылки на свои репозитории с конфигами

⚠️ Важное правило безопасности (актуально именно для тебя как ИБ-специалиста): **никогда не запускай `curl | bash` скрипты с GitHub не глядя**. Сначала открой скрипт, прочитай, что он делает, потом запускай. Это база гигиены, но про неё почему-то часто забывают именно при "риса-инсталлах".

---

## 12. Полезное для ИБ-повседневки

- **BlackArch repo** (опционально, поверх обычного Arch) — добавляет репозиторий с тысячами security-тулз без необходимости ставить отдельную ОС:
  https://blackarch.org/downloads.html — там инструкция по добавлению репозитория через `strap.sh`
- Для самого пентестинга — держи отдельную VM (virt-manager/QEMU или VirtualBox) с Kali/BlackArch, не мешай это с основной системой
- `ufw` или `firewalld` — не забудь поднять файрвол на самой основной машине
- `timeshift` или `snapper` — снапшоты системы перед экспериментами с конфигами, чтобы не пересобирать всё с нуля при поломке

---

## Итоговый чек-лист

- [ ] Бэкап данных с Windows
- [ ] Скачан и проверен ISO Arch
- [ ] Записана флешка (Rufus, DD-режим)
- [ ] Secure Boot выключен, UEFI режим включён
- [ ] Загрузка с флешки, интернет подключён
- [ ] Диск размечен (2 раздела: EFI + корень), отформатирован, смонтирован
- [ ] Файл подкачки (swapfile) создан и активирован
- [ ] (опц.) зеркала ускорены через reflector
- [ ] pacstrap — базовая система установлена, включая `sudo`, `git` и `*-ucode`
- [ ] fstab сгенерирован (включая своп)
- [ ] chroot: локаль, время, хостнейм, пароль root
- [ ] Пользователь создан, sudo настроен
- [ ] systemd-boot установлен и настроен, ucode-строка в arch.conf добавлена
- [ ] Перед umount — swapoff, затем чистая перезагрузка
- [ ] Драйверы GPU, аудио, NetworkManager
- [ ] yay установлен
- [ ] Hyprland + экосистема установлены
- [ ] Дотфайлы с GitHub подключены
