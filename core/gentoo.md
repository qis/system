# Gentoo
Use "Admin CD" from <https://www.gentoo.org/downloads/> to create a memory stick.

```sh
# Change root password.
passwd

# Start SSH service.
rc-service sshd start

# Get IP address.
ip addr
```

SSH into live environment.

```sh
# Log in as root.
ssh root@core

# Confirm UEFI mode.
ls /sys/firmware/efi

# Synchronize time.
chronyd -q 'server 0.gentoo.pool.ntp.org iburst'

# List block devices.
lsblk

# Partition disk.
parted -a optimal /dev/nvme0n1
```

```
unit mib
mklabel gpt
mkpart boot 1 257
mkpart swap linux-swap 257 61697
mkpart root 61697 426684
set 1 boot on
set 2 swap on
print
quit
```

Install system.

```sh
# Create boot filesystem.
mkfs.fat -F32 /dev/nvme0n1p1

# Enable swap filesystem.
mkswap /dev/nvme0n1p2
swapon /dev/nvme0n1p2

# Create root filesystem.
modprobe zfs
zpool create -f -o ashift=12 -o cachefile= -O compression=lz4 -O atime=off -m none -R /mnt/gentoo system /dev/nvme0n1p3

# Create system datasets.
zfs create -o mountpoint=/ system/root
zfs create -o mountpoint=/home system/home
zfs create -o mountpoint=/home/qis -o encryption=aes-256-gcm -o keyformat=passphrase -o keylocation=prompt system/home/qis
zfs create -o mountpoint=/tmp -o compression=off -o sync=disabled system/tmp
zfs create -o mountpoint=/opt -o compression=off system/opt
chmod 1777 /mnt/gentoo/tmp

# Mount boot filesystem.
mkdir /mnt/gentoo/boot
mount -o defaults,noatime /dev/nvme0n1p1 /mnt/gentoo/boot

# Download and extract stage 3 tarball using a mirror from https://www.gentoo.org/downloads/mirrors/.
export stage3=20231210T170356Z
export remote=current-stage3-amd64-nomultilib-systemd
export mirror=https://mirror.yandex.ru/gentoo-distfiles/releases/amd64/autobuilds
curl -L ${mirror}/${remote}/stage3-amd64-nomultilib-systemd-${stage3}.tar.xz -o /mnt/gentoo/stage.tar.xz
tar xpf /mnt/gentoo/stage.tar.xz --xattrs-include='*.*' --numeric-owner -C /mnt/gentoo
rm -f /mnt/gentoo/stage.tar.xz

# Redirect /var/tmp.
rmdir /mnt/gentoo/var/tmp
ln -s ../tmp /mnt/gentoo/var/tmp

# Copy zpool cache.
mkdir /mnt/gentoo/etc/zfs
cp -L /etc/zfs/zpool.cache /mnt/gentoo/etc/zfs/

# Mount virtual filesystems.
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run

# Copy network settings.
cp -L /etc/resolv.conf /mnt/gentoo/etc/

# Configure CPU flags.
echo "*/* `cpuid2cpuflags`" > /mnt/gentoo/etc/portage/package.use/flags

# Chroot into system.
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"

# Generate locale.
tee /etc/locale.gen >/dev/null <<'EOF'
en_US.UTF-8 UTF-8
ru_RU.UTF-8 UTF-8
EOF

locale-gen
source /etc/profile
export PS1="(chroot) ${PS1}"

# Configure mouting points.
tee /etc/fstab >/dev/null <<'EOF'
/dev/nvme0n1p1 /boot vfat defaults,noatime,noauto 0 2
/dev/nvme0n1p2 none  swap sw                      0 0
EOF

# Create make.conf.
tee /etc/portage/make.conf >/dev/null <<'EOF'
# Compiler
CFLAGS="-march=znver3 -O2 -pipe"
CXXFLAGS="${CFLAGS}"
FCFLAGS="${CFLAGS}"
FFLAGS="${CFLAGS}"
MAKEOPTS="-j17"

# Locale
LC_MESSAGES=C
LINGUAS="en en_US"
L10N="en en-US"

# Portage
FEATURES="buildpkg"
GRUB_PLATFORMS="efi-64"
ACCEPT_LICENSE="* -@EULA"
CONFIG_PROTECT="/var/bind"
EMERGE_DEFAULT_OPTS="--with-bdeps=y --keep-going=y --quiet-build=y"
GENTOO_MIRRORS="https://mirror.eu.oneandone.net/linux/distributions/gentoo/gentoo/"
VIDEO_CARDS="fbdev amdgpu radeon"
LUA_SINGLE_TARGET="luajit"

# System
USE="bash-completion caps dbus icu idn -nls opencl policykit -sendmail udev unicode xml"
USE="${USE} -X -gui -gnome -gtk -cairo -pango -kde -kwallet -plasma -qt5 -qml -sdl -xcb"
USE="${USE} -branding -doc -examples -handbook -sound -test -telemetry"
EOF

# Synchronize portage.
cp /usr/share/portage/config/repos.conf /etc/portage/repos.conf
emerge --sync

# Read portage news.
eselect news read

# Create sets directory.
mkdir /etc/portage/sets

# Create @core set.
curl -L https://raw.githubusercontent.com/qis/system/master/core/etc/portage/package.accept_keywords/core \
     -o /etc/portage/package.accept_keywords/core
curl -L https://raw.githubusercontent.com/qis/system/master/core/etc/portage/package.mask/core \
     -o /etc/portage/package.mask/core
curl -L https://raw.githubusercontent.com/qis/system/master/core/etc/portage/package.use/core \
     -o /etc/portage/package.use/core
curl -L https://raw.githubusercontent.com/qis/system/master/core/etc/portage/sets/core \
     -o /etc/portage/sets/core

# Verify that the @world set has no conflicts.
emerge -pve @world

# Install kernel sources.
emerge -s '^sys-kernel/gentoo-sources$'
echo "=sys-kernel/gentoo-sources-6.1.67 ~amd64" > /etc/portage/package.accept_keywords/kernel
echo "=sys-kernel/gentoo-sources-6.1.67 symlink" > /etc/portage/package.use/kernel
emerge -avnuU =sys-kernel/gentoo-sources-6.1.67 sys-kernel/linux-firmware

# Enter kernel directory.
cd /usr/src/linux

# Download kernel configs.
curl -L https://raw.githubusercontent.com/qis/system/master/core/usr/src/linux/core -o .config.core

# Clean kernel directory.
make clean mrproper distclean

# Configure kernel.
cp .config.core .config
make menuconfig

# Build and install kernel.
make -j17
make modules_prepare modules_install install

# Delete old kernel files.
rm -f /boot/*.old

# Rebuild @world set.
emerge -ave @world
emerge -ac

# Install @core set.
emerge -avn @core
emerge -ac

# Enable filesystem services.
systemctl enable zfs.target
systemctl enable zfs-import-cache
systemctl enable zfs-mount
systemctl enable zfs-import.target

# Rebuild kernel modules.
emerge -av @module-rebuild

# Generate kernel initarmfs.
bliss-initramfs -k 6.1.67-gentoo
mv initrd-6.1.67-gentoo /boot/

# Remount NVRAM variables (efivars) with read/write access.
mount -o remount,rw /sys/firmware/efi/efivars/

# Install systemd bootloader.
bootctl install

# Configure systemd bootloader.
tee /boot/loader/loader.conf >/dev/null <<'EOF'
timeout 3
default @saved
console-mode keep
auto-firmware false
auto-entries true
editor false
beep false
EOF

# Create systemd bootloader entry.
tee /boot/loader/entries/linux.conf >/dev/null <<'EOF'
title Linux
linux /vmlinuz-6.1.67-gentoo
initrd /initrd-6.1.67-gentoo
options root=system/root ro acpi_enforce_resources=lax quiet
EOF

# Set default systemd bootloader entry.
bootctl list
bootctl set-default linux.conf

# Install efibootmgr(8) after disabling "Boot Order Lock" in UEFI settings.
emerge -avn sys-boot/efibootmgr

# List boot entries.
efibootmgr

# Delete boot entries.
# efibootmgr -B -b 0
# efibootmgr -B -b 1

# Create boot entry.
efibootmgr -c -d /dev/nvme0n1 -p 1 -L "Linux Boot Manager" -l '\EFI\systemd\systemd-bootx64.efi'

# Configure virtual memory.
mkdir /etc/sysctl.d
tee /etc/sysctl.d/vm.conf >/dev/null <<'EOF'
vm.max_map_count=2147483642
vm.swappiness=1
EOF

# Configure git.
git config --global core.eol lf
git config --global core.autocrlf false
git config --global core.filemode false
git config --global pull.rebase false

# Configure nvim.
git clone --recursive https://github.com/qis/vim /etc/xdg/nvim

# Build nvim telescope fzf plugin.
cd /etc/xdg/nvim/pack/plugins/opt/telescope-fzf-native
cmake -DCMAKE_BUILD_TYPE=Release -B build
cmake --build build

# Create nvim symlinks.
mkdir /root/.config
ln -s ../../etc/xdg/nvim /root/.config/nvim
ln -s nvim /usr/bin/vim

# Configure editor.
echo "EDITOR=/usr/bin/vim" > /etc/env.d/01editor

# Configure time zone.
ln -snf /usr/share/zoneinfo/Europe/Berlin /etc/localtime

# Configure shell.
curl -L https://raw.githubusercontent.com/qis/system/master/conf/bash.sh -o /etc/bash/bashrc.d/core

# Configure tmux.
curl -L https://raw.githubusercontent.com/qis/system/master/conf/tmux.conf -o /etc/tmux.conf

# Configure seatd.
systemctl enable seatd

# Configure ethernet.
systemctl enable sshd
systemctl enable dhcpcd
systemctl enable systemd-networkd
systemctl enable systemd-resolved
tee /etc/systemd/network/eth0.network >/dev/null <<'EOF'
[Match]
Name=eth0

[Network]
DHCP=yes
EOF

# Configure wireless.
emerge -avn net-wireless/iw net-wireless/wpa_supplicant
tee /etc/systemd/network/wlan0.network >/dev/null <<'EOF'
[Match]
Name=wlan0

[Network]
DHCP=yes
EOF

rfkill unblock all

tee /etc/wpa_supplicant/wpa_supplicant-wlan0.conf >/dev/null <<'EOF'
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=wheel
device_name=core
update_config=0
eapol_version=1
fast_reauth=1
country=DE

ap_scan=1
autoscan=exponential:3:60

network={
  priority=1
  scan_ssid=1
  ssid="......."
  psk="........"
  proto=RSN
  key_mgmt=WPA-PSK
  pairwise=CCMP
  group=CCMP
}
EOF

chmod 0640 /etc/wpa_supplicant/wpa_supplicant-wlan0.conf

ln -s /lib/systemd/system/wpa_supplicant@.service \
  /etc/systemd/system/multi-user.target.wants/wpa_supplicant@wlan0.service

systemctl enable wpa_supplicant@wlan0

# Configure sensors and fan.
emerge -avn app-laptop/thinkfan

mkdir /etc/modprobe.d
tee /etc/modprobe.d/thinkpad.conf >/dev/null <<'EOF'
options thinkpad_acpi fan_control=1
EOF

tee /etc/thinkfan.conf >/dev/null <<'EOF'
sensors:
  - tpacpi: /proc/acpi/ibm/thermal
    indices: [0]

fans:
  - tpacpi: /proc/acpi/ibm/fan

levels:
  - ["level auto", 0, 60]
  - [4, 60, 70]
  - [5, 70, 75]
  - [6, 75, 85]
  - [7, 85, 95]
  - ["level full-speed", 95, 32767]
EOF

systemctl enable thinkfan

# Configure power button and lid switch.
tee -a /etc/systemd/logind.conf >/dev/null <<'EOF'
HandlePowerKey=suspend
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
EOF

# Configure pip.
pip config set global.target ~/.pip
tee /etc/profile.d/pip.sh >/dev/null <<'EOF'
export PATH="${PATH}:${HOME}/.pip/bin"
export PYTHONPATH="${HOME}/.pip"
EOF

# Configure npm.
npm config set prefix ~/.npm
tee /etc/profile.d/npm.sh >/dev/null <<'EOF'
export PATH="${PATH}:${HOME}/.npm/bin"
EOF

# Configure application path.
tee /etc/profile.d/path.sh >/dev/null <<'EOF'
export PATH="${HOME}/.local/bin:${PATH}"
EOF

# Configure desktop runtime directory.
tee /etc/profile.d/xdg.sh >/dev/null <<'EOF'
XDG_RUNTIME_DIR=/run/user/${UID}
EOF

# Configure desktop user directories.
emerge -avn x11-misc/xdg-user-dirs
tee /etc/xdg/user-dirs.defaults >/dev/null <<'EOF'
DOWNLOAD=downloads
DOCUMENTS=documents
DESKTOP=.local/desktop
PUBLICSHARE=.local/public
TEMPLATES=.local/templates
PICTURES=documents/pictures
VIDEOS=.local/videos
MUSIC=.local/music
EOF

# Update environment.
env-update
source /etc/profile
export PS1="(chroot) ${PS1}"

# Add user.
useradd -m -G users,seat,wheel,audio,input,usb,video -s /bin/bash qis
chown -R qis:qis /home/qis
passwd qis

# Configure sudo.
EDITOR=tee visudo >/dev/null <<'EOF'
Defaults env_keep += "LANG LANGUAGE LINGUAS LC_* MM_CHARSET _XKB_CHARSET"
Defaults env_keep += "EDITOR PAGER LS_COLORS TERM TMUX SESSION USERPROFILE"

root  ALL=(ALL:ALL) ALL
qis   ALL=(ALL:ALL) NOPASSWD: ALL

@includedir /etc/sudoers.d
EOF

# Create home directory mount script.
tee /sbin/mount-zfs-home >/dev/null <<'EOF'
#!/bin/bash
set -eu

# Receive password via stdin.
PASS=$(cat -)

# List the "local" value of the "canmount" property for each zfs dataset.
zfs get canmount -s local -H -o name,value | while read line; do
  # Filter on canmount == 'noauto'. Filesystems marked 'noauto'
  # can be mounted, but is not done so automatically during boot.
  canmount=$(echo $line | awk '{print $2}')
  [[ $canmount = 'noauto' ]] || continue

  # Filter on user property core.encrypt.automount:user.
  # It should match the user that we are logging in as ($PAM_USER).
  volname=$(echo $line | awk '{print $1}')
  user=$(zfs get core.encrypt.automount:user -s local -H -o value $volname)
  [[ $user = $PAM_USER ]] || continue

  # Unlock and mount the volume.
  zfs load-key "$volname" <<< "$PASS" || continue
  zfs mount "$volname" || true # ignore erros
done
EOF

chmod +x /sbin/mount-zfs-home
echo "auth            optional        pam_exec.so expose_authtok /sbin/mount-zfs-home" >> /etc/pam.d/system-auth

# Set home directory properties.
zfs set canmount=noauto system/home/qis
zfs set core.encrypt.automount:user=qis system/home/qis

# Link resolv.conf to systemd.
ln -snf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf

# Merge config changes.
# Press 'q' to quit pager.
# Press 'n' to skip patch.
dispatch-conf

# Set root password.
passwd

# Exit chroot environment.
exit

# Unmount filesystems.
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo

# Halt system and remove installation media.
halt -p
```

Configure system.

```sh
# Install SSH config.
scp -r .ssh core:

# Log in as user.
ssh core

# Configure ssh.
chmod 0755 .ssh
chmod 0644 .ssh/*
chmod 0600 .ssh/id_rsa
cp .ssh/id_rsa.pub .ssh/authorized_keys
rm -f .ssh/.known_hosts* .ssh/known_hosts

# Log in as root.
sudo su -

# Set hostname.
hostnamectl hostname core

# Configure locale.
localectl list-locales
localectl set-locale LANG=en_US.UTF-8
localectl set-locale LC_TIME=C.UTF-8
localectl set-locale LC_CTYPE=C.UTF-8
localectl set-locale LC_COLLATE=C.UTF-8
localectl set-locale LC_NUMERIC=C.UTF-8
localectl set-locale LC_MESSAGES=C.UTF-8
localectl set-locale LC_PAPER=ru_RU.UTF-8
localectl set-locale LC_NAME=ru_RU.UTF-8
localectl set-locale LC_ADDRESS=ru_RU.UTF-8
localectl set-locale LC_TELEPHONE=ru_RU.UTF-8
localectl set-locale LC_MONETARY=ru_RU.UTF-8
localectl set-locale LC_MEASUREMENT=ru_RU.UTF-8
localectl set-locale LC_IDENTIFICATION=ru_RU.UTF-8

# Update environment.
env-update
source /etc/profile

# Configure systemd.
systemd-firstboot --prompt --setup-machine-id

# Configure systemd services.
systemctl preset-all --preset-mode=enable-only

# Enable time synchronization.
systemctl enable systemd-timesyncd

# Log out as root.
exit
```

## User
Configure user settings.

```sh
# Create user directories.
LC_ALL=C xdg-user-dirs-update --force
mkdir -p ~/.local/bin

# Configure git.
git config --global core.eol lf
git config --global core.autocrlf false
git config --global core.filemode false
git config --global pull.rebase false

# Configure pip.
rm -rf ~/.pip; mkdir ~/.pip
pip config set global.target ~/.pip

# Configure npm.
rm -rf ~/.npm; mkdir ~/.npm
npm config set prefix ~/.npm

# Install npm packages.
npm install -g npm
npm install -g \
  typescript typescript-language-server eslint prettier terser \
  rollup @rollup/plugin-typescript rollup-plugin-terser \
  rollup-plugin-serve rollup-plugin-livereload neovim

# Configure nvim.
git clone --recursive git@github.com:qis/vim ~/.config/nvim

# Build nvim telescope fzf plugin.
cd ~/.config/nvim/pack/plugins/opt/telescope-fzf-native
cmake -DCMAKE_BUILD_TYPE=Release -B build
cmake --build build

# Build nvim treesitter libraries.
nvim
```

```
:TSInstall lua
:TSInstall javascript
:TSInstall typescript
```

```sh
# Reboot system.
sudo reboot
```

## Administration
Common administrative tasks.

```sh
# List ports in world set.
equery l @world

# List world set and dependencies.
emerge -epv @world

# Rebuild world set after sync or cahnged USE flags.
emerge -auUD @world

# Remove port from world set.
emerge -W app-admin/sudo

# Uninstall removed ports.
emerge -ac

# Merge config changes in response to emerge warnings.
# IMPORTANT: config file '...' needs updating.
# Press 'q' to quit pager.
# Press 'n' to skip patch.
dispatch-conf

# Remove build directories.
rm -rf /var/tmp/portage

# List files installed by a port.
equery f --tree app-shells/bash

# List ports that depend on a port.
equery d app-shells/bash

# Show port that installed a specific file.
equery b `which bash`

# Find ports by name.
equery w sudo

# Find ports by file.
e-file nvim

# List hardware.
lshw -c display

# Watch log output.
sudo journalctl -b -n all -f -u thinkfan

# Mount snapshot.
# mount -t zfs system/home/qis@<date> /mnt
```

<details>
<summary>Update</summary>

Update system.

```sh
# Mount boot partition.
mount /boot

# Backup boot partition.
tar cvJf /var/boot-`date '+%F'`.tar.xz -C / boot

# List snapshots.
zfs list -t snapshot

# Destroy old snapshots.
# zfs destroy system/root@<name>
# zfs destroy system/home/qis@<name>

# Create new snapshots.
zfs snapshot system/root@`date '+%F'`
zfs snapshot system/home/qis@`date '+%F'`

# Synchronize portage repositories.
emaint sync -a

# Read portage news.
eselect news read

# Update linux kernel sources.
emerge -s '^sys-kernel/gentoo-sources$'
echo "=sys-kernel/gentoo-sources-X.Y.ZZ ~amd64" > /etc/portage/package.accept_keywords/kernel
echo "=sys-kernel/gentoo-sources-X.Y.ZZ symlink" > /etc/portage/package.use/kernel
emerge -avn =sys-kernel/gentoo-sources-X.Y.ZZ

# Update linux kernel sources symlink.
eselect kernel list
eselect kernel set linux-X.Y.ZZ-gentoo
eselect kernel show
readlink /usr/src/linux

# 1. Rebuild kernels.
# 2. Rebuild kernel modules.
# 3. Rebuild initarmfs.
# 4. Update systemd bootloader entries.

# Reboot system.
reboot

# Check kernel version.
uname -r

# Uninstall old kernel sources.
emerge -W =sys-kernel/gentoo-sources-6.1.67
emerge -ac

# Remove old kernel binaries.
rm -f /boot/*-6.1.67-*

# Remove old kernel modules.
rm -rf /lib/modules/6.1.67-gentoo

# Delete old kernel sources.
rm -rf /usr/src/linux-6.1.67-gentoo

# Reboot system.
reboot

# Update @world set.
emerge -auUD @world
emerge -ac

# Reboot system.
reboot
```

</details>

<details>
<summary>Rescue</summary>

Boot "Admin CD" memory stick.

```sh
passwd
rc-service sshd start

rm ~/.ssh/known_hosts; ssh root@10.0.0.100

swapon /dev/nvme0n1p2 && \
zpool import -fR /mnt/gentoo system && \
mount -o defaults,noatime /dev/nvme0n1p1 /mnt/gentoo/boot && \
mount --types proc /proc /mnt/gentoo/proc && \
mount --rbind /sys /mnt/gentoo/sys && \
mount --make-rslave /mnt/gentoo/sys && \
mount --rbind /dev /mnt/gentoo/dev && \
mount --make-rslave /mnt/gentoo/dev && \
mount --bind /run /mnt/gentoo/run && \
mount --make-slave /mnt/gentoo/run && \
rm -f /mnt/gentoo/etc/resolv.conf && \
cp -L /etc/resolv.conf /mnt/gentoo/etc/ && \
chroot /mnt/gentoo /bin/bash

source /etc/profile

# Perform rescue operations.

# Restore resolv.conf symlink.
ln -snf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
exit

# Unmount filesystems.
umount -l /mnt/gentoo/dev{/shm,/pts,} && \
umount -R /mnt/gentoo && \
halt -p
```

</details>


<details>
<summary>Backup</summary>

Create backup.

```sh
# Set snapshot name.
snapshot=`date '+%F'`

# Mount boot partition.
sudo mount /boot

# Create boot partition archive.
sudo tar cvJf /var/boot-${snapshot}.tar.xz -C / boot

# Create snapshots.
sudo zfs snapshot -r system@${snapshot}

# Send snapshots.
sudo zfs send -Rw system@${snapshot} | ssh -o compression=no core sudo zfs recv system/core

# Destroy snapshots.
sudo zfs destroy -r system@${snapshot}

# Delete boot partition archive.
sudo rm -f /var/boot-${snapshot}.tar.xz

# Connect to backup host.
ssh core

# Fix mount points.
sudo zfs list -o name,mountpoint
sudo zfs set mountpoint=/core system/core/root
sudo zfs set mountpoint=/core/home system/core/home
sudo zfs set mountpoint=/core/home/qis system/core/home/qis
sudo zfs set mountpoint=/core/opt system/core/opt
sudo zfs set mountpoint=/core/opt/data system/core/opt/data
sudo zfs set mountpoint=/core/tmp system/core/tmp
sudo zfs load-key -r system/core/home/qis
sudo zfs mount system/core/home/qis

# Destroy snapshots.
sudo zfs destroy -r system/core@2023-12-12
```

</details>

<!--
# NOTE: The "post" part can fail.

tee /lib/systemd/system-sleep/wifi.sh >/dev/null <<'EOF'
#!/bin/bash
if [ "${1}" = "pre" ]; then
  exec /sbin/rmmod mtk_t7xx
elif [ "${1}" = "post" ]; then
  exec /sbin/modprobe mtk_t7xx
fi
exit 0
EOF

chmod +x /lib/systemd/system-sleep/wifi.sh
-->
