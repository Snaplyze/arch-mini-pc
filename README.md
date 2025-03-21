# Настройка Arch Linux на мини-ПК с Intel Celeron N5095

## Характеристики системы
- Процессор: Intel Celeron N5095 со встроенным графическим ядром Intel
- ОЗУ: 16 ГБ
- Накопители: 2 SATA SSD (системный диск - Btrfs, второй диск для медиа - ext4)
- Рабочий стол: GNOME

## 1. Подготовка системы

### Обновление системы
```bash
# Обновление репозиториев и пакетов
sudo pacman -Syu
```

### Настройка зеркал с помощью reflector
```bash
# Установка reflector
sudo pacman -S reflector

# Создание резервной копии текущего списка зеркал
sudo cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup

# Настройка зеркал для России, сортировка по скорости
sudo reflector --country Russia --latest 10 --protocol https --sort rate --save /etc/pacman.d/mirrorlist

# Проверка обновленного списка зеркал
cat /etc/pacman.d/mirrorlist

# Настройка автоматического обновления зеркал
sudo mkdir -p /etc/xdg/reflector
sudo tee /etc/xdg/reflector/reflector.conf > /dev/null << 'EOF'
--save /etc/pacman.d/mirrorlist
--country Russia
--protocol https
--latest 10
--sort rate
EOF

# Включение и запуск службы reflector
sudo systemctl enable reflector.timer
sudo systemctl start reflector.timer
```

### Установка базовых утилит
```bash
# Установка необходимых утилит для разработки и системного администрирования
sudo pacman -S base-devel git wget curl vim htop neofetch
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
```

### Настройка Xorg для Intel Graphics
```bash
# Создание конфигурационного файла
sudo mkdir -p /etc/X11/xorg.conf.d/
sudo tee /etc/X11/xorg.conf.d/20-intel.conf > /dev/null << 'EOF'
Section "Device"
    Identifier  "Intel Graphics"
    Driver      "intel"
    Option      "TearFree" "true"
    Option      "AccelMethod" "sna"
    Option      "DRI" "3"
EndSection
EOF
```

## 3. Оптимизация системы

### Настройка управления энергопотреблением

### Настройка управления энергопотреблением

#### Выбор системы управления энергопотреблением
В Arch Linux есть две основные системы управления энергопотреблением: TLP и Power Profiles Daemon (PPD). Они конфликтуют между собой, поэтому нужно выбрать один из вариантов:

**Вариант 1: Использование TLP (больше настроек, но без графического интерфейса)**
```bash
# Установка TLP и дополнительных инструментов
sudo pacman -S tlp tlp-rdw

# Установка инструментов для мониторинга температуры
sudo pacman -S lm_sensors
sudo sensors-detect --auto

# Настройка конфигурации TLP для Celeron N5095
sudo tee /etc/tlp.conf > /dev/null << 'EOF'
# Основные настройки
TLP_ENABLE=1
TLP_DEFAULT_MODE=AC
TLP_PERSISTENT_DEFAULT=0

# Настройки CPU для Intel Celeron
CPU_SCALING_GOVERNOR_ON_AC=performance
CPU_SCALING_GOVERNOR_ON_BAT=powersave
CPU_ENERGY_PERF_POLICY_ON_AC=performance
CPU_ENERGY_PERF_POLICY_ON_BAT=power
CPU_MIN_PERF_ON_AC=0
CPU_MAX_PERF_ON_AC=100
CPU_MIN_PERF_ON_BAT=0
CPU_MAX_PERF_ON_BAT=80

# Настройки для Intel GPU
INTEL_GPU_MIN_FREQ_ON_AC=300
INTEL_GPU_MAX_FREQ_ON_AC=1100
INTEL_GPU_MIN_FREQ_ON_BAT=300
INTEL_GPU_MAX_FREQ_ON_BAT=800
INTEL_GPU_BOOST_FREQ_ON_AC=1100
INTEL_GPU_BOOST_FREQ_ON_BAT=800

# Настройки SATA
SATA_LINKPWR_ON_AC="max_performance"
SATA_LINKPWR_ON_BAT="medium_power"

# Настройки звука
SOUND_POWER_SAVE_ON_AC=0
SOUND_POWER_SAVE_ON_BAT=1

# Настройки PCI Express
PCIE_ASPM_ON_AC=performance
PCIE_ASPM_ON_BAT=powersave
EOF

# Активация TLP
sudo systemctl enable tlp.service
sudo systemctl start tlp.service

# Проверка статуса TLP
sudo tlp-stat -s

# Вы можете установить утилиту управления TLP с графическим интерфейсом
paru -S tlpui
```

**Вариант 2: Использование Power Profiles Daemon (интегрирован в GNOME, простой графический интерфейс)**
```bash
# Установка Power Profiles Daemon для управления режимами питания через интерфейс GNOME
sudo pacman -S power-profiles-daemon

# Убедитесь, что TLP не установлен или отключен, чтобы избежать конфликтов
sudo systemctl disable tlp.service --now || true
sudo pacman -R tlp tlp-rdw || true

# Установка lm_sensors для мониторинга температуры
sudo pacman -S lm_sensors
sudo sensors-detect --auto

# Активация PPD
sudo systemctl enable power-profiles-daemon.service
sudo systemctl start power-profiles-daemon.service

# Установка GNOME расширения для удобного управления профилями питания (необязательно)
paru -S gnome-shell-extension-power-profile-switcher

# Проверка статуса Power Profiles Daemon
powerprofilesctl list
```

**Вариант 3: Использование обоих с ограничением TLP (не рекомендуется, но возможно)**
```bash
# Установка обоих
sudo pacman -S tlp power-profiles-daemon

# Настройка TLP так, чтобы он не конфликтовал с PPD
sudo mkdir -p /etc/tlp.d/
sudo tee /etc/tlp.d/99-disable-pm.conf > /dev/null << 'EOF'
# Отключаем управление CPU и GPU в TLP, чтобы избежать конфликтов с PPD
CPU_SCALING_GOVERNOR_ON_AC=""
CPU_SCALING_GOVERNOR_ON_BAT=""
CPU_SCALING_MIN_FREQ_ON_AC=0
CPU_SCALING_MAX_FREQ_ON_AC=0
CPU_SCALING_MIN_FREQ_ON_BAT=0
CPU_SCALING_MAX_FREQ_ON_BAT=0
CPU_ENERGY_PERF_POLICY_ON_AC=""
CPU_ENERGY_PERF_POLICY_ON_BAT=""
CPU_BOOST_ON_AC=0
CPU_BOOST_ON_BAT=0
PLATFORM_PROFILE_ON_AC=""
PLATFORM_PROFILE_ON_BAT=""
# Отключаем Intel GPU настройки
INTEL_GPU_MIN_FREQ_ON_AC=0
INTEL_GPU_MAX_FREQ_ON_AC=0
INTEL_GPU_MIN_FREQ_ON_BAT=0
INTEL_GPU_MAX_FREQ_ON_BAT=0
INTEL_GPU_BOOST_FREQ_ON_AC=0
INTEL_GPU_BOOST_FREQ_ON_BAT=0
EOF

# Активация обоих сервисов
sudo systemctl enable tlp.service
sudo systemctl start tlp.service
sudo systemctl enable power-profiles-daemon.service
sudo systemctl start power-profiles-daemon.service
```

Для стандартного использования рекомендуется **Вариант 2 (Power Profiles Daemon)**, так как он хорошо интегрирован с GNOME и предоставляет простой графический интерфейс для переключения профилей производительности.


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

### Настройка swappiness
```bash
# Уменьшение swappiness для лучшей производительности на системах с достаточным ОЗУ
sudo tee /etc/sysctl.d/99-swappiness.conf > /dev/null << 'EOF'
vm.swappiness=10
EOF

# Применение изменений
sudo sysctl -p /etc/sysctl.d/99-swappiness.conf
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

### Настройка ZRAM для улучшения производительности
```bash
# Включение ZRAM для улучшения производительности с ограниченной памятью
sudo pacman -S zram-generator
sudo tee /etc/systemd/zram-generator.conf > /dev/null << 'EOF'
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
EOF
sudo systemctl restart systemd-zram-setup@zram0.service
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

### Оптимизация сетевых настроек
```bash
# Оптимизация сетевого стека
sudo tee /etc/sysctl.d/99-network-performance.conf > /dev/null << 'EOF'
# Увеличение размера буфера приема и отправки
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.rmem_default = 1048576
net.core.wmem_default = 1048576
net.ipv4.tcp_rmem = 4096 1048576 16777216
net.ipv4.tcp_wmem = 4096 1048576 16777216

# Оптимизация TCP
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_mtu_probing = 1
EOF

# Применение изменений
sudo sysctl --system
```

## 4. Настройка системы Btrfs и резервного копирования

### Настройка Snapper для Btrfs
```bash
# Инструменты для работы с Btrfs
sudo pacman -S snapper snap-pac grub-btrfs

# Настройка Snapper
sudo umount /.snapshots || true  # Игнорировать ошибку, если не примонтировано
sudo rm -rf /.snapshots || true
sudo snapper -c root create-config /
sudo mkdir -p /.snapshots
sudo chmod 750 /.snapshots

# Настройка Snapper для регулярных снапшотов
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer
```

### Установка Timeshift
```bash
# Установка Timeshift для создания резервных копий
paru -S timeshift

# Теперь можно запустить Timeshift и настроить его:
sudo timeshift-launcher
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

## 6. Установка современной аудиосистемы

### Установка PipeWire
```bash
# Установка PipeWire вместо PulseAudio для лучшей производительности
sudo pacman -S pipewire pipewire-pulse pipewire-alsa pipewire-jack wireplumber

# Установка утилит для управления звуком
sudo pacman -S pavucontrol easyeffects

# Переключение на PipeWire
systemctl --user enable pipewire.service pipewire-pulse.service wireplumber.service
systemctl --user start pipewire.service pipewire-pulse.service wireplumber.service

# Отключение PulseAudio, если он установлен
systemctl --user disable pulseaudio.service pulseaudio.socket
systemctl --user stop pulseaudio.service pulseaudio.socket
```

## 7. Настройка Bluetooth

### Установка и активация Bluetooth
```bash
# Установка необходимых пакетов для Bluetooth
sudo pacman -S bluez bluez-utils bluez-tools

# Установка графических утилит для Bluetooth
sudo pacman -S blueman

# Включение и запуск службы Bluetooth
sudo systemctl enable bluetooth.service
sudo systemctl start bluetooth.service

# Настройка автоматического включения Bluetooth при загрузке
sudo tee /etc/bluetooth/main.conf > /dev/null << 'EOF'
[General]
# Настройка автоматического включения
AutoEnable=true
# Улучшение качества звука для A2DP
ControllerMode = bredr
EOF

# Перезапуск службы для применения изменений
sudo systemctl restart bluetooth.service

# Добавление текущего пользователя в группу bluetooth
sudo usermod -aG lp $USER
```

### Интеграция Bluetooth с PipeWire
```bash
# Убедимся, что Bluetooth аудио работает с PipeWire
sudo pacman -S libspa-bluetooth

# Проверка статуса Bluetooth
bluetoothctl show
```

## 8. Установка Steam и игровых компонентов

### Включение мультибиблиотечной поддержки
```bash
# Включение multilib репозитория для 32-битных приложений
sudo sed -i "/\[multilib\]/,/Include/s/^#//" /etc/pacman.conf
sudo pacman -Syu
```

### Установка Steam
```bash
# Установка Steam
sudo pacman -S steam

# Установка дополнительных библиотек для Steam
sudo pacman -S lib32-mesa lib32-nvidia-utils lib32-libpulse lib32-alsa-plugins
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

## 9. Установка дополнительных программ и утилит

### Мультимедийные кодеки и проигрыватели
```bash
# Установка мультимедийных кодеков
sudo pacman -S ffmpeg gst-plugins-base gst-plugins-good gst-plugins-bad gst-plugins-ugly gst-libav

# Установка мультимедийных проигрывателей
sudo pacman -S vlc mpv celluloid
```

### Инструменты разработки
```bash
# Установка популярных языков программирования и инструментов
sudo pacman -S python python-pip nodejs npm gcc make cmake

# Установка редакторов кода
sudo pacman -S code # Visual Studio Code
# ИЛИ
paru -S visual-studio-code-bin # Если предпочитаете бинарную версию

# Установка инструментов для разработки
sudo pacman -S git-lfs github-cli meld

# Конфигурация Git
git config --global user.name "Ваше Имя"
git config --global user.email "ваша.почта@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase false
git config --global core.editor "vim"
```

### Офисные и интернет-приложения
```bash
# Браузеры
sudo pacman -S firefox

# Установка Google Chrome
paru -S google-chrome

# Офисные приложения
sudo pacman -S libreoffice-fresh

# Утилиты для работы с архивами
sudo pacman -S p7zip unrar unzip zip
```

### Системные мониторы и утилиты
```bash
# Установка улучшенных системных мониторов
sudo pacman -S htop btop bpytop

# Улучшенный терминал и оболочка
sudo pacman -S zsh
paru -S oh-my-zsh-git

# Настройка zsh как оболочки по умолчанию (при желании)
chsh -s /usr/bin/zsh

# Улучшенные утилиты командной строки
sudo pacman -S bat fd ripgrep fzf rsync

# Сетевые инструменты
sudo pacman -S iftop nethogs nmap traceroute
```

## 10. Оптимизация GNOME

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

## 11. Настройка безопасности

### Настройка брандмауэра
```bash
# Установка и настройка UFW (простой брандмауэр)
sudo pacman -S ufw gufw  # gufw - графический интерфейс для UFW
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

# Установка и настройка Fail2Ban для защиты от брутфорс-атак
sudo pacman -S fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
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

## 12. Настройка автоматических обновлений

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

## 13. Полезные советы и обслуживание системы

### Ускорение загрузки Arch Linux
```bash
# Отключение ненужных сервисов
systemctl --user mask evolution-addressbook-factory.service evolution-calendar-factory.service evolution-source-registry.service

# Анализ времени загрузки
systemd-analyze blame
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

### Проверка системы
```bash
# Проверка на наличие ошибок в журнале
sudo journalctl -p 3 -b

# Проверка температуры процессора
sensors

# Проверка использования дисков
df -h

# Проверка состояния Btrfs
sudo btrfs scrub status /
sudo btrfs fi usage /
```

## Заключение

Данная инструкция охватывает все основные аспекты настройки Arch Linux на мини-ПК с Intel Celeron N5095. После выполнения всех шагов вы получите полностью оптимизированную систему с:

- Настроенными драйверами Intel для графики и процессора
- Оптимизированной файловой системой Btrfs для системного диска
- Настроенной системой резервного копирования и снапшотов
- Установленными Docker, Docker Compose для разработки
- Настроенным Steam с Proton GE для игр
- Современной аудиосистемой на базе PipeWire
- Настроенным и оптимизированным Bluetooth
- Улучшенной безопасностью и производительностью
- Оптимизированными сетевыми настройками

Регулярно обновляйте систему командой `paru -Syu` для поддержания безопасности и стабильности работы.

Дополнительно рекомендуется периодически проверять состояние Btrfs файловой системы с помощью:
```bash
sudo btrfs scrub status /
```

И следить за свободным местом:
```bash
sudo btrfs fi usage /
```
