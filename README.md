# Настройка Arch Linux на мини-ПК с Intel Celeron N5095

## Характеристики системы
- Процессор: Intel Celeron N5095 со встроенным графическим ядром Intel
- ОЗУ: 16 ГБ
- Накопители: 2 SATA SSD (системный диск - Btrfs, второй диск для медиа - ext4)
- Рабочий стол: GNOME с Wayland
- Аудиосистема: PipeWire

## 1. Подготовка системы

### Обновление системы
```bash
# Обновление репозиториев и пакетов
sudo pacman -Syu
```

### Установка базовых утилит
```bash
# Установка необходимых утилит для разработки и системного администрирования
sudo pacman -S base-devel git wget curl vim htop neofetch
```

### Настройка Reflector для оптимизации зеркал
```bash
# Установка Reflector для выбора оптимальных зеркал
sudo pacman -S reflector

# Создание резервной копии списка зеркал
sudo cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup

# Настройка Reflector для выбора 20 наиболее быстрых зеркал
sudo reflector --verbose --latest 20 --sort rate --save /etc/pacman.d/mirrorlist

# Настройка автоматического обновления зеркал
sudo mkdir -p /etc/xdg/reflector/
sudo tee /etc/xdg/reflector/reflector.conf > /dev/null << 'EOF'
--save /etc/pacman.d/mirrorlist
--protocol https
--country "Russia,Belarus,Germany,France"
--latest 20
--sort rate
EOF

# Включение и запуск службы Reflector
sudo systemctl enable reflector.timer
sudo systemctl start reflector.timer
```

### Настройка AUR-помощника (paru)
```bash
# Установка paru для удобной работы с AUR в скрытую папку .paru
mkdir -p ~/.paru
git clone https://aur.archlinux.org/paru-bin.git ~/.paru/repo
cd ~/.paru/repo
makepkg -si
cd ~

# Настройка paru для использования скрытой папки
mkdir -p ~/.config/paru
tee ~/.config/paru/paru.conf > /dev/null << 'EOF'
[options]
BottomUp
SudoLoop
CleanAfter
Provides
PgpFetch
Devel
Upgrademenu
BatchInstall
EOF
```

## 2. Установка и настройка драйверов

### Графические драйверы Intel
```bash
# Установка драйверов Intel для встроенной графики
sudo pacman -S mesa intel-media-driver libva-intel-driver intel-gpu-tools

# Установка микрокода Intel для улучшения стабильности
sudo pacman -S intel-ucode

# Дополнительные пакеты для оптимальной работы Wayland
sudo pacman -S xorg-server-xwayland qt5-wayland qt6-wayland xorg-mkfontscale xorg-fonts-cyrillic xorg-fonts-misc
```

### Настройка Wayland для Intel Graphics
```bash
# Создание конфигурационного файла для улучшения производительности Wayland с Intel
sudo mkdir -p /etc/environment.d/
sudo tee /etc/environment.d/gpu.conf > /dev/null << 'EOF'
# Улучшение производительности на Intel GPU
LIBVA_DRIVER_NAME=iHD
VDPAU_DRIVER=va_gl
# Включение аппаратного ускорения
MOZ_WEBRENDER=1
MOZ_ACCELERATED=1
# Для лучшей поддержки Wayland в приложениях
MOZ_ENABLE_WAYLAND=1
# Для Chrome/Electron приложений
ELECTRON_OZONE_PLATFORM_HINT=wayland
EOF
```

### Установка утилит для управления энергопотреблением
```bash
# Установка power-profiles-daemon для интеграции с GNOME
sudo pacman -S power-profiles-daemon
sudo systemctl enable power-profiles-daemon.service
sudo systemctl start power-profiles-daemon.service

# Установка инструментов для мониторинга температуры
sudo pacman -S lm_sensors
sudo sensors-detect --auto

# Проверка доступных профилей энергопотребления
powerprofilesctl list
```

## 3. Оптимизация системы

### Настройка планировщика дисков
```bash
# Проверяем текущий планировщик для каждого диска
cat /sys/block/sda/queue/scheduler
cat /sys/block/sdb/queue/scheduler

# Настраиваем на использование BFQ для SSD
sudo tee /etc/udev/rules.d/60-ioschedulers.rules > /dev/null << 'EOF'
# Set scheduler for SSD
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="bfq"
EOF
```

### Оптимизация использования SSD
```bash
# Настройка TRIM для SSD
sudo systemctl enable fstrim.timer
sudo systemctl start fstrim.timer

# Проверка работы TRIM
sudo fstrim -av
```

### Оптимизация производительности файловой системы
```bash
# Резервная копия fstab
sudo cp /etc/fstab /etc/fstab.backup

# Оптимизация для основного диска с Btrfs
# Добавление опций монтирования для Btrfs (системный диск)
sudo sed -i '/btrfs/ s/defaults/defaults,noatime,compress=zstd:1,space_cache=v2,ssd,discard=async/g' /etc/fstab

# Оптимизация для второго диска с ext4 (для медиа)
sudo sed -i '/ext4/ s/defaults/defaults,noatime,commit=60/g' /etc/fstab

# Проверка изменений
cat /etc/fstab
```

### Настройка swappiness
```bash
# Уменьшение swappiness для лучшей производительности на системах с достаточным ОЗУ
sudo tee /etc/sysctl.d/99-swappiness.conf > /dev/null << 'EOF'
vm.swappiness=10
EOF

# Применение изменений
sudo sysctl -p /etc/sysctl.d/99-swappiness.conf
```

### Настройка системных лимитов
```bash
# Увеличение лимитов системы для разработки
sudo tee /etc/security/limits.conf > /dev/null << 'EOF'
*               soft    nofile          65535
*               hard    nofile          65535
*               soft    nproc           65535
*               hard    nproc           65535
EOF
```

## 4. Установка и настройка PipeWire

### Установка и настройка PipeWire
```bash
# Установка PipeWire и связанных компонентов
sudo pacman -S pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber

# Установка дополнительных инструментов для работы с аудио
sudo pacman -S pavucontrol easyeffects helvum

# Включение сервисов PipeWire
systemctl --user enable pipewire.socket
systemctl --user enable pipewire-pulse.socket
systemctl --user enable wireplumber.service

# Запуск сервисов PipeWire
systemctl --user start pipewire.socket
systemctl --user start pipewire-pulse.socket
systemctl --user start wireplumber.service

# Проверка статуса PipeWire
pactl info | grep "Server Name"
```

### Мультимедийные кодеки и дополнительные библиотеки
```bash
# Установка мультимедийных кодеков
sudo pacman -S ffmpeg gst-plugins-base gst-plugins-good gst-plugins-bad gst-plugins-ugly gst-libav

# Установка дополнительных библиотек для мультимедиа
sudo pacman -S alsa-utils sof-firmware bluez bluez-utils
sudo systemctl enable bluetooth.service
sudo systemctl start bluetooth.service
```

### Тестирование и оптимизация PipeWire
```bash
# Проверка задержки звука
pw-top

# Настройка низкой задержки для PipeWire
mkdir -p ~/.config/pipewire/pipewire.conf.d/
tee ~/.config/pipewire/pipewire.conf.d/99-lowlatency.conf > /dev/null << 'EOF'
context.properties = {
    default.clock.rate = 48000
    default.clock.quantum = 32
    default.clock.min-quantum = 32
    default.clock.max-quantum = 32
}
EOF

# Проверка конфигурации
systemctl --user restart pipewire.service
systemctl --user status pipewire.service

# Для игр через Steam - лучшая совместимость с PipeWire
echo 'SDL_AUDIODRIVER=pipewire' >> ~/.bashrc
```

## 5. Установка Docker и Docker Compose

### Установка Docker
```bash
# Установка Docker
sudo pacman -S docker

# Настройка автозапуска Docker
sudo systemctl enable docker.service
sudo systemctl start docker.service

# Добавление текущего пользователя в группу docker
sudo usermod -aG docker $USER
```

### Установка Docker Compose
```bash
# Установка Docker Compose
sudo pacman -S docker-compose

# Проверка установки
docker-compose --version
```

### Настройка Docker
```bash
# Создание конфигурационного файла для Docker
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

# Перезапуск Docker для применения настроек
sudo systemctl restart docker
```

## 6. Установка Steam и игровых компонентов

### Включение мультибиблиотечной поддержки
```bash
# Включение multilib репозитория для 32-битных приложений
sudo sed -i "/\[multilib\]/,/Include/s/^#//" /etc/pacman.conf
sudo pacman -Syu
```

### Установка Steam и Wayland-совместимость
```bash
# Установка Steam
sudo pacman -S steam

# Установка дополнительных библиотек для Steam
sudo pacman -S lib32-mesa lib32-pipewire lib32-pipewire-jack lib32-alsa-plugins

# Создание файла для запуска Steam в режиме Wayland
mkdir -p ~/.config/steam
tee ~/.config/steam/steam-wayland.sh > /dev/null << 'EOF'
#!/bin/bash
SDL_VIDEODRIVER=wayland
steam -forcedesktopscaling 1 -fulldesktopres "$@"
EOF
chmod +x ~/.config/steam/steam-wayland.sh

# Создание .desktop файла для запуска Steam в Wayland
mkdir -p ~/.local/share/applications/
tee ~/.local/share/applications/steam-wayland.desktop > /dev/null << 'EOF'
[Desktop Entry]
Name=Steam (Wayland)
Comment=Application for managing and playing games on Steam
Exec=~/.config/steam/steam-wayland.sh %U
Icon=steam
Terminal=false
Type=Application
Categories=Network;FileTransfer;Game;
MimeType=x-scheme-handler/steam;x-scheme-handler/steamlink;
EOF
```

### Установка Proton GE (для лучшей совместимости с Windows-играми)
```bash
# Создание каталога для Proton GE
mkdir -p ~/.steam/root/compatibilitytools.d/

# Загрузка последней версии Proton GE
VERSION=$(curl -s https://api.github.com/repos/GloriousEggroll/proton-ge-custom/releases/latest | grep "tag_name" | cut -d\" -f4)
wget https://github.com/GloriousEggroll/proton-ge-custom/releases/download/$VERSION/$VERSION.tar.gz

# Распаковка архива в каталог Proton
tar -xf $VERSION.tar.gz -C ~/.steam/root/compatibilitytools.d/
rm $VERSION.tar.gz
```

### Установка Gamemode (для оптимизации игр)
```bash
# Установка Gamemode
sudo pacman -S gamemode lib32-gamemode

# Проверка установки
gamemoded -t
```

## 7. Установка дополнительных библиотек и программ

### Инструменты разработки
```bash
# Установка популярных языков программирования и инструментов
sudo pacman -S python python-pip nodejs npm gcc make cmake

# Установка редакторов кода
sudo pacman -S code # Visual Studio Code
# ИЛИ
paru -S visual-studio-code-bin # Если предпочитаете бинарную версию
```

### Офисные и интернет-приложения
```bash
# Браузеры
sudo pacman -S firefox

# Установка Google Chrome
paru -S google-chrome

# Настройка Chrome для Wayland
mkdir -p ~/.config/chrome-flags.d/
tee ~/.config/chrome-flags.d/wayland.conf > /dev/null << 'EOF'
--enable-features=UseOzonePlatform
--ozone-platform=wayland
EOF

# Офисные приложения
sudo pacman -S libreoffice-fresh

# Утилиты для работы с архивами
sudo pacman -S p7zip unrar unzip zip
```

## 8. Оптимизация GNOME для Wayland

### Настройка GNOME для оптимальной работы с Wayland
```bash
# Установка дополнительных компонентов для Wayland
sudo pacman -S xdg-desktop-portal xdg-desktop-portal-gnome gnome-keyring

# Настройка GNOME для использования Wayland по умолчанию
sudo tee /etc/gdm/custom.conf > /dev/null << 'EOF'
# GDM configuration
[daemon]
# Uncomment the line below to force the login screen to use Xorg
#WaylandEnable=false

[security]

[xdmcp]

[chooser]

[debug]
EOF

# Дополнительные настройки для Wayland
gsettings set org.gnome.mutter experimental-features "['scale-monitor-framebuffer']"

# Настройка плагина для управления питанием в GNOME
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.desktop.session idle-delay 900
```

### Установка дополнительных расширений GNOME
```bash
# Установка GNOME Tweaks и GNOME Extensions
sudo pacman -S gnome-tweaks gnome-shell-extensions

# Установка Extension Manager для удобного управления расширениями
paru -S extension-manager
```

### Рекомендуемые расширения GNOME
1. Dash to Dock или Dash to Panel
2. GSConnect (интеграция с Android)
3. Caffeine (предотвращение автоматического отключения экрана)
4. Resource Monitor (мониторинг ресурсов)

Расширения можно установить через [extensions.gnome.org](https://extensions.gnome.org) или через приложение Extension Manager.

### Настройка внешнего вида
```bash
# Установка тем и иконок
sudo pacman -S papirus-icon-theme arc-gtk-theme arc-icon-theme

# Настройка шрифтов
sudo pacman -S ttf-dejavu ttf-liberation noto-fonts noto-fonts-emoji ttf-roboto ttf-roboto-mono
```

## 9. Настройка автоматических обновлений

### Установка и настройка системы автообновления
```bash
# Установка пакета для автоматических обновлений
sudo pacman -S pacman-contrib

# Создание службы и таймера для автоматических обновлений
sudo tee /etc/systemd/system/pacman-updates.service > /dev/null << 'EOF'
[Unit]
Description=Pacman database update

[Service]
Type=oneshot
ExecStart=/usr/bin/pacman -Sy
EOF

sudo tee /etc/systemd/system/pacman-updates.timer > /dev/null << 'EOF'
[Unit]
Description=Update pacman database daily

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
EOF

# Активация таймера
sudo systemctl enable pacman-updates.timer
sudo systemctl start pacman-updates.timer
```

## 10. Завершающие настройки

### Настройка брандмауэра
```bash
# Установка и настройка UFW (простой брандмауэр)
sudo pacman -S ufw
sudo systemctl enable ufw
sudo systemctl start ufw

# Настройка базовых правил
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https

# Включение брандмауэра
sudo ufw enable
```

### Создание резервных копий системы
```bash
# Установка Timeshift для создания резервных копий
paru -S timeshift

# Запуск Timeshift и настройка резервного копирования
sudo timeshift-launcher
```

### Проверка системы
```bash
# Проверка на наличие ошибок в журнале
sudo journalctl -p 3 -b

# Проверка температуры процессора
sensors

# Проверка использования дисков
df -h
```

## 11. Полезные советы

### Решение проблем с Wayland
```bash
# Проверка, запущена ли сессия в Wayland
echo $XDG_SESSION_TYPE

# Диагностика Wayland
journalctl -b -p 3

# Для приложений, которые не поддерживают Wayland, создайте скрипт-обертку
mkdir -p ~/bin
tee ~/bin/xwayland-run > /dev/null << 'EOF'
#!/bin/bash
GDK_BACKEND=x11 QT_QPA_PLATFORM=xcb "$@"
EOF
chmod +x ~/bin/xwayland-run

# Добавление ~/bin в PATH, если его еще нет
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Использование: xwayland-run <программа>
```

### Решение проблем с аудио через PipeWire
```bash
# Если возникают проблемы с аудио в Steam или Wine
sudo pacman -S lib32-pipewire lib32-pipewire-jack

# Если нужна совместимость с JACK-приложениями
sudo pacman -S realtime-privileges
sudo usermod -aG realtime $USER

# Для диагностики проблем с PipeWire
pw-cli dump
pw-cli info all

# Для исправления проблем с Bluetooth-аудио
paru -S pipewire-bluetooth-a2dp-codecs-git
systemctl --user restart pipewire-pulse.service
```

### Ускорение загрузки Arch Linux
```bash
# Отключение ненужных сервисов
systemctl --user mask evolution-addressbook-factory.service evolution-calendar-factory.service evolution-source-registry.service

# Анализ времени загрузки
systemd-analyze blame

# Отключение службы NetworkManager-wait-online.service для ускорения загрузки
sudo systemctl disable NetworkManager-wait-online.service

# Оптимизация параметров загрузки в GRUB
sudo sed -i 's/GRUB_TIMEOUT=5/GRUB_TIMEOUT=1/' /etc/default/grub
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### Очистка системы
```bash
# Очистка кэша pacman
sudo pacman -Sc

# Удаление ненужных зависимостей
sudo pacman -Rns $(pacman -Qtdq)

# Очистка кэша paru
paru -Sc

# Дополнительные оптимизации для Btrfs
sudo btrfs balance start -dusage=85 /
sudo btrfs scrub start /
```

### Обновление микропрограмм
```bash
# Установка fwupd для обновления микропрограмм
sudo pacman -S fwupd

# Получение доступных обновлений микропрограмм
sudo fwupdmgr refresh
sudo fwupdmgr get-updates

# Установка обновлений микропрограмм
sudo fwupdmgr update
```

### Оптимизация процессора Intel и управление энергопотреблением
```bash
# Установка утилит для управления процессором Intel
sudo pacman -S intel-undervolt thermald

# Настройка thermald для контроля температуры процессора
sudo systemctl enable thermald
sudo systemctl start thermald

# Создание конфигурации для intel-undervolt (требуется осторожность)
sudo cp /etc/intel-undervolt.conf /etc/intel-undervolt.conf.bak
sudo nano /etc/intel-undervolt.conf

# После настройки включите сервис
sudo systemctl enable intel-undervolt
sudo systemctl start intel-undervolt

# Настройка доступа к режимам энергосбережения через GNOME
gsettings set org.gnome.settings-daemon.plugins.power power-saver-profile-on-low-battery true

# Проверка режимов производительности
powerprofilesctl list
```

## Заключение

Данная инструкция охватывает полный процесс настройки Arch Linux на мини-ПК с Intel Celeron N5095. После выполнения всех шагов вы получите оптимизированную систему с:

- Оптимизированными зеркалами через Reflector
- Установленным помощником AUR (paru)
- Настроенными драйверами Intel и Wayland
- Оптимизированными файловыми системами (Btrfs и ext4)
- Современной аудиосистемой PipeWire
- Установленными Docker и Docker Compose
- Настроенным Steam с поддержкой Wayland
- Google Chrome и другими полезными приложениями
- Оптимизированным рабочим столом GNOME

Регулярно обновляйте систему командой `sudo pacman -Syu` для поддержания безопасности и стабильности работы.

Для полного применения всех изменений рекомендуется перезагрузить систему после завершения настройки:

```bash
sudo reboot
```
