# Debian Bookworm on the Samsung SM-P355 / Galaxy Tab A 8.0 (MSM8916)
### A Complete Step-by-Step Guide

---

## What You Will End Up With

Debian bookworm (arm64) on the **Samsung Galaxy Tab A 8.0 (2015)**, model **SM-P355**, codename **gt58**, powered by the **Qualcomm Snapdragon 410 (MSM8916)** SoC.

The finished system:

```
OS:      Debian GNU/Linux 12 (bookworm) aarch64
Host:    Samsung Galaxy Tab A 8.0 (2015)
Kernel:  Linux 6.12.1-msm8916+
GPU:     Freedreno FD307 (hardware accelerated)
CPU:     msm8916 (4) @ 1.00 GHz
RAM:     1.83 GiB
Disk:    SD card (ext4)
WiFi:    wlan0 via wcn36xx, NetworkManager
Audio:   samsung-a2015 ALSA card
Display: DSI-1 768x1024 @ 60 Hz
```

Everything from the kernel down — lk2nd, msm-firmware-loader, rmtfs, WiFi, audio, GPU — is identical to the Arch guide. Only the rootfs setup and package management change.

**Development machine:** Lenovo ThinkPad T470 running Arch Linux, cross-compiling as user `zuzu`.

---

## Prerequisites

### Hardware
- Samsung Galaxy Tab A 8.0 / SM-P355 (gt58)
- MicroSD card, 32 GB or larger
- USB cable (Micro-USB)
- A PC running Linux

### Software on your PC

**On Arch:**
```bash
sudo pacman -S debootstrap qemu-user-static binfmt-support \
  base-devel git android-tools wget aarch64-linux-gnu-gcc
```

**On Debian/Ubuntu:**
```bash
sudo apt install debootstrap qemu-user-static binfmt-support \
  build-essential git android-tools-fastboot wget \
  gcc-aarch64-linux-gnu
```

---

## Part 1 — The Boot Chain

Identical to the Arch guide. Nothing changes here.

```
Samsung bootloader → lk2nd → mainline Linux kernel → Debian bookworm
```

The Android firmware partitions on the internal eMMC are left untouched. `msm-firmware-loader` reads from them at boot.

---

## Part 2 — Flashing lk2nd

If lk2nd is already flashed from your Arch install, skip this entirely.

If starting fresh, download the latest MSM8916 image from:

```
https://github.com/msm8916-mainline/lk2nd/releases
```

Boot the tablet into stock fastboot mode (hold **Volume Down + Power** while plugging in USB), then:

```bash
fastboot flash boot lk2nd-msm8916.img
fastboot reboot
```

From now on **Volume Down + Power** enters lk2nd's fastboot mode.

---

## Part 3 — Preparing the SD Card

Identical to the Arch guide. Identify your SD card with `lsblk`.

```bash
sudo fdisk /dev/sdX
# Partition 1: ~512 MB  → /boot
# Partition 2: rest     → /
```

```bash
sudo mkfs.ext4 /dev/sdX1
sudo mkfs.ext4 /dev/sdX2
```

```bash
sudo mkdir -p /mnt/boot /mnt/root
sudo mount /dev/sdX1 /mnt/boot
sudo mount /dev/sdX2 /mnt/root
```

---

## Part 4 — Installing Debian bookworm

### Bootstrap the rootfs

`debootstrap` builds a minimal Debian system directly into the SD card root partition. Since your PC is x86_64, it needs QEMU to run arm64 binaries in the chroot.

```bash
sudo debootstrap \
  --arch=arm64 \
  --foreign \
  bookworm \
  /mnt/root \
  https://deb.debian.org/debian
```

The `--foreign` flag stages the bootstrap without executing anything — execution happens inside the chroot via QEMU.

### Copy QEMU into the chroot

```bash
sudo cp /usr/bin/qemu-aarch64-static /mnt/root/usr/bin/
```

### Complete the bootstrap

```bash
sudo chroot /mnt/root /debootstrap/debootstrap --second-stage
```

This takes a few minutes.

### Enter the chroot

```bash
sudo mount --bind /dev      /mnt/root/dev
sudo mount --bind /proc     /mnt/root/proc
sudo mount --bind /sys      /mnt/root/sys
sudo mount --bind /dev/pts  /mnt/root/dev/pts
sudo chroot /mnt/root /bin/bash
```

### Configure the base system

```bash
# Hostname
echo "gt58" > /etc/hostname

# Hosts file
cat > /etc/hosts <<EOF
127.0.0.1   localhost
127.0.1.1   gt58
::1         localhost ip6-localhost ip6-loopback
EOF

# Timezone
ln -sf /usr/share/zoneinfo/Europe/Budapest /etc/localtime
# Adjust the above to your timezone

# Locale
apt install -y locales
dpkg-reconfigure locales
# Select en_US.UTF-8 or your preferred locale

# Root password
passwd root
```

### Configure apt sources

```bash
cat > /etc/apt/sources.list <<EOF
deb https://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb https://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
deb https://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
EOF

apt update
```

### Install essential packages

```bash
apt install -y \
  systemd systemd-sysv udev dbus \
  sudo \
  networkmanager \
  openssh-server \
  wget curl git nano \
  linux-firmware \
  alsa-utils alsa-ucm-conf \
  xorg xserver-xorg-input-libinput \
  lightdm \
  icewm \
  onboard \
  mesa-vulkan-drivers libgl1-mesa-dri \
  xrandr xinput \
  fastfetch \
  build-essential meson ninja-build pkg-config \
  libglib2.0-dev libudev-dev libqrtr-glib-dev
```

### Create your user

```bash
adduser zuzu
# Follow the prompts, set a password

# Add to necessary groups
usermod -aG sudo,video,input,audio,netdev zuzu
```

### Configure LightDM

LightDM is the display manager. Configure it to use onboard as the on-screen keyboard and optionally autologin:

```bash
cat > /etc/lightdm/lightdm.conf <<EOF
[Seat:*]
greeter-session=lightdm-gtk-greeter
autologin-user=zuzu
autologin-user-timeout=0
EOF
```

Install the GTK greeter:

```bash
apt install -y lightdm-gtk-greeter lightdm-gtk-greeter-settings
```

Configure the greeter to show the on-screen keyboard:

```bash
cat > /etc/lightdm/lightdm-gtk-greeter.conf <<EOF
[greeter]
keyboard=onboard
EOF
```

### Enable services

```bash
systemctl enable NetworkManager
systemctl enable lightdm
systemctl enable ssh
```

### Exit the chroot

```bash
exit
sudo umount /mnt/root/dev/pts /mnt/root/dev /mnt/root/proc /mnt/root/sys
```

---

## Part 5 — Building the Mainline Kernel

Identical to the Arch guide. Use the msm8916-mainline kernel at `v6.12.1-msm8916` with the pmaports config.

```bash
git clone https://github.com/msm8916-mainline/linux.git \
  --depth=1 --branch v6.12.1-msm8916 ~/linux
cd ~/linux

git clone https://gitlab.postmarketos.org/postmarketOS/pmaports.git ~/pmaports

cp ~/pmaports/device/testing/linux-postmarketos-qcom-msm8916/config-postmarketos-qcom-msm8916.aarch64 .config

export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
make olddefconfig
make -j$(nproc) Image.gz dtbs modules
```

### Install modules to the SD card

```bash
sudo mount /dev/sdX2 /mnt/root
sudo make INSTALL_MOD_PATH=/mnt/root modules_install
```

### Copy kernel and DTB

```bash
sudo mount /dev/sdX1 /mnt/boot
sudo cp arch/arm64/boot/Image.gz /mnt/boot/
sudo cp arch/arm64/boot/dts/qcom/msm8916-samsung-gt58.dtb /mnt/boot/
```

---

## Part 6 — Boot Configuration

Identical to the Arch guide.

```bash
sudo mkdir -p /mnt/boot/extlinux
sudo nano /mnt/boot/extlinux/extlinux.conf
```

```
LABEL Debian bookworm
  LINUX /Image.gz
  FDT /msm8916-samsung-gt58.dtb
  APPEND root=/dev/mmcblk1p2 rootwait rw rootfstype=ext4 console=tty0 loglevel=7
```

### Build boot.img

```bash
cd ~/linux

cat arch/arm64/boot/Image.gz \
    arch/arm64/boot/dts/qcom/msm8916-samsung-gt58.dtb > Image-dtb.gz

mkbootimg \
  --kernel Image-dtb.gz \
  --cmdline "root=/dev/mmcblk1p2 rootwait rw rootfstype=ext4 console=tty0 loglevel=7" \
  --base 0x80000000 \
  --pagesize 2048 \
  --os_version 11.0 \
  --os_patch_level 2025-01 \
  -o boot.img
```

### Test and flash

```bash
fastboot boot boot.img      # test first
fastboot flash boot boot.img  # flash once confirmed
```

---

## Part 7 — Firmware Loading (msm-firmware-loader)

Identical to the Arch guide. The script is pure shell and runs fine on Debian.

Create the script:

```bash
sudo nano /usr/sbin/msm-firmware-loader.sh
```

Paste the full script (same as Arch guide, Part 7). Then:

```bash
sudo chmod +x /usr/sbin/msm-firmware-loader.sh
```

Create the service:

```bash
sudo nano /usr/lib/systemd/system/msm-firmware-loader.service
```

```ini
[Unit]
Description=Load firmware that is located on dedicated partitions of qcom devices
Before=systemd-udevd.service
DefaultDependencies=no
RequiresMountsFor=/sys /dev

[Service]
ExecStart=/usr/sbin/msm-firmware-loader.sh
Type=oneshot
RemainAfterExit=yes

[Install]
WantedBy=sysinit.target
```

Create the run directory:

```bash
sudo mkdir -p /run/msm-firmware-loader
echo "d /run/msm-firmware-loader 0755 root root -" | \
  sudo tee /etc/tmpfiles.d/msm-firmware-loader.conf

sudo systemctl enable msm-firmware-loader.service
```

---

## Part 8 — USB RNDIS Gadget (SSH During Initial Setup)

Identical to the Arch guide. Enable only when needed before WiFi is configured.

```bash
sudo systemctl enable usb-gadget.service
sudo systemctl start usb-gadget.service
```

Configure `usb0` via systemd-networkd:

```bash
sudo nano /etc/systemd/network/80-usb0.network
```

```ini
[Match]
Name=usb0

[Network]
Address=192.168.7.1/24
DHCPServer=yes

[DHCPServer]
PoolOffset=10
PoolSize=20
```

```bash
sudo systemctl enable systemd-networkd
```

From your PC:

```bash
sudo ip addr add 192.168.7.2/24 dev usb0
sudo ip link set usb0 up
ssh zuzu@192.168.7.1
```

Share internet to the tablet:

```bash
# On your PC
sudo sysctl net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
sudo iptables -A FORWARD -i usb0 -j ACCEPT
```

On the tablet:

```bash
ip route add default via 192.168.7.2
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

---

## Part 9 — WiFi

Identical to the Arch guide. NetworkManager manages `wlan0` automatically once the WCNSS firmware is loaded.

```bash
nmtui
# or
nmcli device wifi connect "YourSSID" password "YourPassword"
```

Disable the USB gadget once WiFi works:

```bash
sudo systemctl disable usb-gadget.service
```

---

## Part 10 — rmtfs

`rmtfs` is not packaged in Debian. Build it from source on the tablet (requires internet via USB tethering or WiFi first):

```bash
# Install build dependencies (already installed if you followed Part 4)
sudo apt install -y meson ninja-build pkg-config \
  libglib2.0-dev libudev-dev libqrtr-glib-dev

git clone https://github.com/andersson/rmtfs.git
cd rmtfs
meson setup build
ninja -C build
sudo ninja -C build install
```

Enable the service:

```bash
sudo systemctl enable rmtfs
```

---

## Part 11 — Audio

Identical to the Arch guide. The audio stack is hardware-level and distro-independent.

### Load audio modules at boot

```bash
sudo nano /etc/modules-load.d/audio.conf
```

```
apr
pdr_interface
q6core
q6afe
q6afe-clocks
q6afe-dai
q6asm
q6asm-dai
q6adm
q6routing
q6voice
q6voice-dai
snd-soc-msm8916-analog
snd-soc-msm8916-digital
snd-soc-tfa989x
snd-soc-apq8016-sbc
```

### UCM configuration

Check if Debian's `alsa-ucm-conf` package includes the `samsung-a2015` profile:

```bash
ls /usr/share/alsa/ucm2/ | grep samsung
```

If it is missing, copy the UCM files from pmaports as with the Arch guide. The msm8916-mainline project also maintains UCM configs at:

```
https://github.com/msm8916-mainline/alsa-ucm-conf
```

### Verify audio

After a full boot:

```bash
cat /proc/asound/cards
# 0 [samsunga2015]: samsung-a2015

alsaucm -c 0 set _verb HiFi
speaker-test -D hw:0,2 -c 1 -t wav
```

---

## Part 12 — Display and GPU

Identical to the Arch guide. Freedreno FD307 works with hardware acceleration, no software rendering flags needed.

Verify:

```bash
glxinfo | grep renderer
# renderer: FD307
```

---

## Part 13 — Touchscreen

Identical to the Arch guide. The Zinitix driver is in-kernel and libinput picks it up automatically. No config needed in default portrait orientation.

For display rotation:

```bash
sudo nano /etc/X11/xorg.conf.d/99-display.conf
```

```
Section "Monitor"
    Identifier "DSI-1"
    Option "Rotate" "right"
EndSection

Section "InputClass"
    Identifier "Rotated Touchscreen"
    MatchIsTouchscreen "on"
    Driver "libinput"
    Option "TransformationMatrix" "0 1 0 -1 0 1 0 0 1"
EndSection
```

---

## Part 14 — Screen Brightness

Identical to the Arch guide.

```bash
echo 100 | sudo tee /sys/class/backlight/1a98000.dsi.0/brightness
```

Allow your user to set it without sudo:

```bash
sudo nano /etc/udev/rules.d/90-backlight.rules
```

```
ACTION=="add", SUBSYSTEM=="backlight", \
  RUN+="/bin/chmod a+w /sys/class/backlight/%k/brightness"
```

---

## Debian-Specific Notes

### Sandboxing

The main reason to switch from ALARM. Debian bookworm ships conservative, well-tested package versions and its kernel configuration expectations (seccomp, user namespaces, cgroups) are better matched by the mainline kernel config from pmaports than ALARM's rolling setup. Flatpak, Chromium sandboxing, and bubblewrap should work without the namespace errors common on ALARM.

If you still hit sandbox issues, check:

```bash
# Verify user namespaces are enabled
cat /proc/sys/kernel/unprivileged_userns_clone
# Should be 1; if 0:
echo 1 | sudo tee /proc/sys/kernel/unprivileged_userns_clone

# Make it permanent
echo "kernel.unprivileged_userns_clone=1" | \
  sudo tee /etc/sysctl.d/99-namespaces.conf
```

### Package management

Standard apt. No AUR, no manual PKGBUILDs. Packages that required AUR on ALARM (like `rmtfs`) need building from source once, but everything else is just `apt install`.

### Kernel updates

Unlike ALARM which pushes kernel updates through pacman, on Debian **you manage the kernel yourself** — it is not in apt at all. When you want to update the kernel, repeat Part 5 of this guide (pull the new tag, build, copy to SD card, rebuild boot.img). This is actually an advantage for stability — the kernel only changes when you decide it does.

### fstab

Add proper entries so the filesystems mount correctly:

```bash
sudo nano /etc/fstab
```

```
/dev/mmcblk1p2  /       ext4    defaults,noatime  0 1
/dev/mmcblk1p1  /boot   ext4    defaults          0 2
```

---

## Troubleshooting

### `debootstrap` fails during second stage

Make sure `binfmt-support` is running and QEMU is registered:

```bash
sudo systemctl restart binfmt-support
sudo update-binfmts --enable qemu-aarch64
```

Then retry:

```bash
sudo chroot /mnt/root /debootstrap/debootstrap --second-stage
```

### LightDM does not start

```bash
journalctl -u lightdm --no-pager
```

Common cause: missing `xserver-xorg-video-fbdev` or `xserver-xorg-video-modesetting`. Install both:

```bash
sudo apt install -y xserver-xorg-video-modesetting xserver-xorg-video-fbdev
```

### Sandboxing still broken after switching from ALARM

Check unprivileged user namespaces as described above. Also verify the kernel was built with:

```
CONFIG_USER_NS=y
CONFIG_SECCOMP=y
CONFIG_SECCOMP_FILTER=y
CONFIG_CGROUPS=y
```

These are all set in the pmaports config.

### WiFi, audio, remoteproc issues

All identical to the Arch guide troubleshooting section — these are hardware-level, not distro-level.

### `rmtfs` build fails: `libqrtr-glib not found`

```bash
sudo apt install -y libqrtr-glib-dev
```

If not available in bookworm repos:

```bash
git clone https://gitlab.freedesktop.org/mobile-broadband/libqrtr-glib.git
cd libqrtr-glib
meson setup build
ninja -C build
sudo ninja -C build install
```

Then rebuild rmtfs.

---

## Boot Sequence Summary

```
Power on
  → Samsung/Qualcomm bootloader
  → lk2nd (internal eMMC boot partition)
  → reads extlinux.conf from SD card /boot
  → Linux 6.12.1-msm8916+ starts
  → systemd
      → msm-firmware-loader.service
            mounts apnhlos, modem, persist from internal eMMC
            symlinks blobs into /run/msm-firmware-loader/target/
            sets firmware search path
      → remoteproc1 (WCNSS → wlan0)
      → remoteproc0 (modem → audio DSP)
      → rmtfs.service
      → NetworkManager (WiFi connects)
      → lightdm (Xorg on vt7)
            user logs in
            → icewm + onboard
```

---

## Resources

- [msm8916-mainline project](https://github.com/msm8916-mainline) — kernel, lk2nd, msm-firmware-loader
- [pmaports — linux-postmarketos-qcom-msm8916](https://gitlab.postmarketos.org/postmarketOS/pmaports/-/tree/master/device/testing/linux-postmarketos-qcom-msm8916) — kernel config
- [Debian bookworm release notes](https://www.debian.org/releases/bookworm/)
- [rmtfs](https://github.com/andersson/rmtfs)
- [Freedreno / Mesa docs](https://docs.mesa3d.org/drivers/freedreno.html)
