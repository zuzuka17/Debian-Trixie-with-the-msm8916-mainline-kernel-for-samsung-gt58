# Debian Trixie on the Samsung SM-P355 / Galaxy Tab A 8.0 (MSM8916)
### A Complete Step-by-Step Guide

---

## What You Will End Up With

```
OS:      Debian GNU/Linux 13 (trixie) aarch64
Host:    Samsung Galaxy Tab A 8.0 (2015)
Kernel:  Linux 7.1-msm8916+
GPU:     Freedreno FD307 (hardware accelerated)
CPU:     msm8916 (4) @ 1.00 GHz
RAM:     1.83 GiB
Disk:    SD card (ext4)
WiFi:    wlan0 via wcn36xx, NetworkManager
Audio:   samsung-a2015 ALSA card
Display: DSI-1 768x1024 @ 60 Hz
```

**Development machine:** Lenovo ThinkPad T470 running Arch Linux, cross-compiling as user `zuzu`.

---

## Prerequisites

### Software on your PC

```bash
sudo pacman -S debootstrap qemu-user-static binfmt-support \
  base-devel git android-tools wget aarch64-linux-gnu-gcc
```

---

## Part 1 — Flash lk2nd

Skip if already done.

Download the latest MSM8916 image from `https://github.com/msm8916-mainline/lk2nd/releases`, boot the tablet into stock fastboot (hold **Volume Down + Power** while plugging in USB), then:

```bash
fastboot flash boot lk2nd-msm8916.img
fastboot reboot
```

From now on **Volume Down + Power** enters lk2nd's fastboot mode.

---

## Part 2 — Prepare the SD Card

Identify your card with `lsblk` — make sure you have the right device before proceeding.

```bash
sudo fdisk /dev/sdX
# Partition 1: ~512 MB  → /boot
# Partition 2: rest     → /

sudo mkfs.ext4 /dev/sdX1
sudo mkfs.ext4 /dev/sdX2

sudo mkdir -p /mnt/boot /mnt/root
sudo mount /dev/sdX1 /mnt/boot
sudo mount /dev/sdX2 /mnt/root
```

---

## Part 3 — Bootstrap Debian Trixie

```bash
sudo debootstrap \
  --arch=arm64 \
  --foreign \
  trixie \
  /mnt/root \
  http://deb.debian.org/debian
```

Use `http` not `https` here — `ca-certificates` isn't installed yet inside the chroot so https will fail. You switch to https after the first `apt install`.

Copy QEMU and complete the second stage:

```bash
sudo cp /usr/bin/qemu-aarch64-static /mnt/root/usr/bin/
sudo chroot /mnt/root /debootstrap/debootstrap --second-stage
```

---

## Part 4 — Configure the System Inside the Chroot

### Enter the chroot

```bash
sudo mount --bind /dev      /mnt/root/dev
sudo mount --bind /proc     /mnt/root/proc
sudo mount --bind /sys      /mnt/root/sys
sudo mount --bind /dev/pts  /mnt/root/dev/pts
sudo chroot /mnt/root /bin/bash
```

### Set time, hostname, hosts

The chroot has no clock. Set the date manually so apt doesn't complain about certificate validity:

```bash
date -s "2026-06-05 14:00:00"   # adjust to today's date

echo "gt58" > /etc/hostname

cat > /etc/hosts <<EOF
127.0.0.1   localhost
127.0.1.1   gt58
::1         localhost ip6-localhost ip6-loopback
EOF
```

### Bootstrap apt — http first, then switch to https

```bash
# Sources with http first (no ca-certificates yet)
cat > /etc/apt/sources.list <<EOF
deb http://deb.debian.org/debian trixie main contrib non-free non-free-firmware
deb http://deb.debian.org/debian trixie-updates main contrib non-free non-free-firmware
deb http://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
EOF

apt update
apt install -y ca-certificates

# Switch to https now that ca-certificates is installed
cat > /etc/apt/sources.list <<EOF
deb https://deb.debian.org/debian trixie main contrib non-free non-free-firmware
deb https://deb.debian.org/debian trixie-updates main contrib non-free non-free-firmware
deb https://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
EOF

apt update
apt full-upgrade -y
```

### Set timezone and locale

```bash
ln -sf /usr/share/zoneinfo/Europe/Budapest /etc/localtime
# Adjust to your timezone

apt install -y locales
/usr/sbin/dpkg-reconfigure locales
# Select en_US.UTF-8 or your preferred locale
```

### Set root password

```bash
passwd root
```

### Install all packages

```bash
apt install -y \
  systemd systemd-sysv systemd-timesyncd udev dbus sudo \
  network-manager openssh-server \
  wget curl git nano \
  firmware-linux alsa-utils alsa-ucm-conf \
  xorg xserver-xorg-input-libinput \
  xserver-xorg-video-fbdev xserver-xorg-video-modesetting \
  lightdm lightdm-gtk-greeter lightdm-gtk-greeter-settings \
  icewm onboard \
  mesa-vulkan-drivers libgl1-mesa-dri \
  x11-xserver-utils xinput \
  fastfetch \
  build-essential meson ninja-build pkg-config \
  libglib2.0-dev libudev-dev libqrtr-glib-dev
```

### Create your user

```bash
adduser zuzu
usermod -aG sudo,video,input,audio,netdev zuzu
```

### Configure LightDM

```bash
cat > /etc/lightdm/lightdm.conf <<EOF
[Seat:*]
greeter-session=lightdm-gtk-greeter
autologin-user=zuzu
autologin-user-timeout=0
EOF

cat > /etc/lightdm/lightdm-gtk-greeter.conf <<EOF
[greeter]
keyboard=onboard
EOF
```

### fstab

```bash
cat > /etc/fstab <<EOF
/dev/mmcblk1p2  /       ext4    defaults,noatime  0 1
/dev/mmcblk1p1  /boot   ext4    defaults          0 2
EOF
```

### Enable services

```bash
systemctl enable NetworkManager
systemctl enable lightdm
systemctl enable ssh
systemctl enable systemd-timesyncd
```

### Exit the chroot

```bash
exit
sudo umount /mnt/root/dev/pts /mnt/root/dev /mnt/root/proc /mnt/root/sys
```

---

## Part 5 — msm-firmware-loader

This reads the proprietary Qualcomm firmware blobs directly from the original Android partitions on the tablet's internal eMMC at boot, and sets them as the kernel firmware search path. Nothing is extracted manually — the Android partitions are left untouched and mounted read-only.

### Create the script

```bash
sudo nano /mnt/root/usr/sbin/msm-firmware-loader.sh
```

Paste the following exactly:

```sh
#!/bin/sh
# SPDX-License-Identifier: MIT
#
# This script is responsible for loading firmware blobs from firmware
# partitions on qcom devices. It will make a dir in tmp, mount all of the
# interesting partitions there and then symlink blobs to a single dir that can
# be then provided to the kernel. (At this time only single additional
# directory can be provided)
#
# This script attempts to load everything at runtime and be as generic
# as possible between the target devices: It should allow a single rootfs
# to be used on multiple different devices as long as all the blobs
# are present on dedicated partitions.
# (Usually the case, Samsung devices ship all blobs, other devices may miss
# venus but that still allows for WiFi and modem to work)

# Get the slot suffix for A/B devices.
# If qbootctl is available query the active slot using that, otherwise rely on
# the kernel cmdline to contain the slot suffix.
# On non-A/B devices the return value will be empty.
# https://source.android.com/docs/core/architecture/bootloader/updating#slots
ab_get_slot() {
	if command -v qbootctl > /dev/null; then
		ab_slot_suffix=$(qbootctl -a | grep -o 'Active slot: _[ab]' | cut -d ":" -f2 | xargs) || :
	else
		ab_slot_suffix=$(grep -o 'androidboot\.slot_suffix=..' /proc/cmdline |  cut -d "=" -f2) || :
	fi
	echo "$ab_slot_suffix"
}

# Configurations:

# List of partitions to be mounted and inspected for blobs.
FW_PARTITIONS="
	apnhlos
	bluetooth$(ab_get_slot)
	dsp$(ab_get_slot)
	modem$(ab_get_slot)
	persist
	vendor$(ab_get_slot)
"

# List of partitions to mount dynamic partitions from.
SUPER_PARTITIONS="
	super
	system
"

# Base directory to be used to unfold the partitions into.
BASEDIR="/run/msm-firmware-loader"

# Preparations:
# This script is intended to run before udev. This means that writeable fs
# may not be available yet. Since this script only creates symlinks, it
# uses tmpfs to work around the early-run limitations as well as to reduce
# disk wear slightly.
mount -o mode=755,nodev,noexec,nosuid -t tmpfs none "$BASEDIR"

mkdir -p "$BASEDIR/mnt"
mkdir -p "$BASEDIR/target"

# Scanning and mounting partitions we're interested in:

# Modern android devices use dynamic partitions for the system.
# To gather firmware from such partitions, search for a "super"
# or "system" partition, and if one is present, map it and try
# to locate firmware partitions of interest inside.
if command -v make-dynpart-mappings > /dev/null
then
	for part in /sys/block/mmcblk*/mmcblk*p* /sys/block/sd*/sd*
	do
		if ! [ -e "$part" ]; then continue; fi;

		DEVNAME="$(grep DEVNAME "$part"/uevent | sed 's/DEVNAME=//g')"
		PARTNAME="$(grep PARTNAME "$part"/uevent | sed 's/PARTNAME=//g')"

		if [ -z "${SUPER_PARTITIONS##*"$PARTNAME"*}" ] && [ -n "$PARTNAME" ]
		then
			if ! make-dynpart-mappings "/dev/$DEVNAME"; then continue; fi;

			for dynpart in /dev/mapper/*
			do
				PARTNAME="$(basename "$dynpart")"
				if [ -z "${FW_PARTITIONS##*"$PARTNAME"*}" ] && [ -n "$PARTNAME" ]
				then
					mkdir -p "$BASEDIR/mnt/$PARTNAME"
					mount -o ro,nodev,noexec,nosuid \
						"$dynpart" "$BASEDIR/mnt/$PARTNAME"
				fi
			done

			break
		fi
	done
fi

# /dev/disk/by-partlabel symlinks don't exist yet, scan sysfs for names instead
for part in /sys/block/mmcblk*/mmcblk*p* /sys/block/sd*/sd*
do
	if ! [ -e "$part" ]; then continue; fi;

	DEVNAME="$(grep DEVNAME "$part"/uevent | sed 's/DEVNAME=//g')"
	PARTNAME="$(grep PARTNAME "$part"/uevent | sed 's/PARTNAME=//g')"

	if [ -z "${FW_PARTITIONS##*"$PARTNAME"*}" ] && [ -n "$PARTNAME" ] && [ ! -d "$BASEDIR/mnt/$PARTNAME" ]
	then
		mkdir -p "$BASEDIR/mnt/$PARTNAME"
		mount -o ro,nodev,noexec,nosuid \
			"/dev/$DEVNAME" "$BASEDIR/mnt/$PARTNAME"
	fi
done

# Linking blobs from all partitions:

EXTRA_PATH="$(cat /sys/module/firmware_class/parameters/path)"

if [ -d "$EXTRA_PATH" ]
then
	for blob in "$EXTRA_PATH"/*
	do
		if ! [ -e "$blob" ]; then break; fi
		ln -s "$blob" "$BASEDIR/target/$(basename "$blob")"
	done
fi

for blob in "$BASEDIR"/mnt/*/image/* "$BASEDIR"/mnt/*/firmware/*
do
	if ! [ -e "$blob" ]; then continue; fi;

	DIR="$(dirname "$blob")"
	BLOBBASE="${blob##*/}"
	BLOBBASE="${BLOBBASE%.*}"

	for prefix in "$BASEDIR/target/$BLOBBASE."*
	do
		if [ -e "$prefix" ]; then continue 2; fi
	done

	for part in "$DIR"/"$BLOBBASE"*
	do
		if [ -f "$part" ]
		then
			ln -s "$part" "$BASEDIR/target/$(basename "$part")"
		fi
	done
done

# Check for sns.reg in persist partition
if [ -f "$BASEDIR"/mnt/persist/sensors/sns.reg ]
then
	mkdir -p "$BASEDIR/target/qcom/sensors"
	ln -s "$BASEDIR"/mnt/persist/sensors/sns.reg "$BASEDIR"/target/qcom/sensors/sns.reg
fi

# Check WCNSS_qcom_wlan_nv.bin in persist partition
if [ -f "$BASEDIR"/mnt/persist/WCNSS_qcom_wlan_nv.bin ]
then
	ln -s "$BASEDIR"/mnt/persist/WCNSS_qcom_wlan_nv.bin "$BASEDIR"/target/WCNSS_qcom_wlan_nv.bin
fi

# venus fixup
if [ -f "$BASEDIR/target/venus.mdt" ] && ! [ -d "$BASEDIR/target/qcom/venus-x" ]
then
	mkdir -p "$BASEDIR/target/qcom/venus-x"
	for part in "$BASEDIR"/target/venus.*
	do
		ln -s "$part" "$BASEDIR/target/qcom/venus-x/$(basename "$part")"
	done
fi

VENUS_DIRS="
	venus-1.8
	venus-3.0
	venus-4.2
	venus-4.4
	venus-5.2
	venus-5.4
	vpu-1.0
	vpu-2.0
"

for vdir in $VENUS_DIRS
do
	if ! [ -d "$BASEDIR/target/qcom/$vdir" ] && [ -f "$BASEDIR/target/venus.mdt" ]
	then
		ln -s "$BASEDIR/target/qcom/venus-x" \
			"$BASEDIR/target/qcom/$vdir"
	fi
done

# WCNSS_qcom_wlan_nv.bin relocation
if [ -h "$BASEDIR"/target/WCNSS_qcom_wlan_nv.bin ]
then
	if ! [ -f "$BASEDIR"/target/wlan/prima/WCNSS_qcom_wlan_nv.bin ]
	then
		mkdir -p "$BASEDIR"/target/wlan/prima
		ln -s "$BASEDIR"/target/WCNSS_qcom_wlan_nv.bin "$BASEDIR"/target/wlan/prima/
	fi
fi

# Bluetooth firmware (ath10k wcn3990 devices)
if [ -d "$BASEDIR/mnt/bluetooth$(ab_get_slot)" ]
then
	mkdir -p "$BASEDIR"/target/qca
	for btblob in "$BASEDIR/mnt/bluetooth$(ab_get_slot)/image"/*
	do
		ln -s "$btblob" "$BASEDIR"/target/qca/
	done
fi

# Symlink .mdt → .mbn so the kernel can autodetect the type
find "$BASEDIR"/target/ \
	-name '*.mdt' \
	-exec sh -c 'ln -s $0 ${0%.mdt}.mbn' {} \;

# Device-model-specific firmware prefix
FIRMWARE_PREFIX=$(find /sys/firmware/devicetree -name "firmware-name" | head -n1 | xargs cat | xargs dirname)

if [ -n "$FIRMWARE_PREFIX" ]
then
	mkdir -p "$BASEDIR/target/$(dirname "$FIRMWARE_PREFIX")"
	ln -s "$BASEDIR/target" "$BASEDIR/target/$FIRMWARE_PREFIX"
fi

# Set the firmware search path
printf "%s" "$BASEDIR/target" > /sys/module/firmware_class/parameters/path
```

Make it executable:

```bash
sudo chmod +x /mnt/root/usr/sbin/msm-firmware-loader.sh
```

### Create the run directory

```bash
sudo mkdir -p /mnt/root/run/msm-firmware-loader
echo "d /run/msm-firmware-loader 0755 root root -" | \
  sudo tee /mnt/root/etc/tmpfiles.d/msm-firmware-loader.conf
```

### Create the systemd service

```bash
sudo tee /mnt/root/usr/lib/systemd/system/msm-firmware-loader.service <<EOF
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
EOF
```

Enable it via chroot:

```bash
sudo chroot /mnt/root systemctl enable msm-firmware-loader.service
```

---

## Part 6 — USB RNDIS Gadget

For SSH access during initial setup before WiFi is working. Disable once WiFi is configured.

### Create the gadget script

```bash
sudo tee /mnt/root/usr/bin/usb-gadget.sh <<'EOF'
#!/bin/sh
modprobe libcomposite
cd /sys/kernel/config/usb_gadget/ || exit 1
mkdir -p msm8916
cd msm8916
echo 0x04E8 > idVendor
echo 0x685D > idProduct
mkdir -p strings/0x409
echo "Samsung"      > strings/0x409/manufacturer
echo "Galaxy Tab A" > strings/0x409/product
echo "0123456789"   > strings/0x409/serialnumber
mkdir -p functions/rndis.usb0
mkdir -p configs/c.1/strings/0x409
echo "RNDIS" > configs/c.1/strings/0x409/configuration
ln -sf functions/rndis.usb0 configs/c.1/
ls /sys/class/udc | head -1 > UDC
EOF

sudo chmod +x /mnt/root/usr/bin/usb-gadget.sh
```

### Create the service

```bash
sudo tee /mnt/root/usr/lib/systemd/system/usb-gadget.service <<EOF
[Unit]
Description=USB RNDIS Gadget
After=msm-firmware-loader.service
Before=NetworkManager.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/usb-gadget.sh

[Install]
WantedBy=multi-user.target
EOF
```

### Configure usb0 networking

```bash
sudo mkdir -p /mnt/root/etc/systemd/network

sudo tee /mnt/root/etc/systemd/network/80-usb0.network <<EOF
[Match]
Name=usb0

[Network]
Address=192.168.7.1/24
DHCPServer=yes

[DHCPServer]
PoolOffset=10
PoolSize=20
EOF
```

Enable via chroot:

```bash
sudo chroot /mnt/root systemctl enable usb-gadget.service
sudo chroot /mnt/root systemctl enable systemd-networkd
```

### Connect from your PC

```bash
sudo ip addr add 192.168.7.2/24 dev usb0
sudo ip link set usb0 up
ssh zuzu@192.168.7.1
```

### Share internet to the tablet over USB

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

## Part 7 — Build the Kernel

Use the msm8916-mainline kernel at `wip/msm8916/7.1-rc6` (or the latest 7.1 tag) with the pmaports config.

```bash
git clone https://github.com/msm8916-mainline/linux.git \
  --depth=1 --branch wip/msm8916/7.1-rc6 ~/linux
cd ~/linux

# Get the pmaports config if you haven't already
git clone https://gitlab.postmarketos.org/postmarketOS/pmaports.git ~/pmaports

cp ~/pmaports/device/testing/linux-postmarketos-qcom-msm8916/config-postmarketos-qcom-msm8916.aarch64 .config

export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
make olddefconfig
make -j$(nproc) Image.gz dtbs modules
```

### Install modules

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

## Part 8 — Boot Configuration

```bash
sudo mkdir -p /mnt/boot/extlinux
sudo tee /mnt/boot/extlinux/extlinux.conf <<EOF
LABEL Debian Trixie
  LINUX /Image.gz
  FDT /msm8916-samsung-gt58.dtb
  APPEND root=/dev/mmcblk1p2 rootwait rw rootfstype=ext4 console=tty0 loglevel=7
EOF
```

### Build and flash boot.img

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

fastboot boot boot.img        # test first
fastboot flash boot boot.img  # flash once confirmed
```

---

## Part 9 — WiFi

NetworkManager manages `wlan0` automatically once the WCNSS firmware is loaded by `msm-firmware-loader`. On first boot connect via:

```bash
nmtui
# or
nmcli device wifi connect "YourSSID" password "YourPassword"
```

Once WiFi works, disable the USB gadget:

```bash
sudo systemctl disable usb-gadget.service
```

---

## Part 10 — rmtfs

Not packaged in Debian. Build from source on the tablet once you have internet (WiFi or USB tethering):

```bash
git clone https://github.com/andersson/rmtfs.git
cd rmtfs
meson setup build
ninja -C build
sudo ninja -C build install
sudo systemctl enable rmtfs
sudo systemctl start rmtfs
```

---

## Part 11 — Audio

The audio stack is identical to the Arch guide — hardware-level, distro-independent.

### Load modules at boot

```bash
sudo tee /etc/modules-load.d/audio.conf <<EOF
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
EOF
```

### UCM configuration

Check if trixie's `alsa-ucm-conf` includes the `samsung-a2015` profile:

```bash
ls /usr/share/alsa/ucm2/ | grep samsung
```

If missing, copy the UCM files from pmaports onto the tablet:

```bash
# On your ThinkPad, copy from pmaports to the tablet over SCP
scp -r ~/pmaports/... zuzu@192.168.7.1:~/ # check pmaports for UCM path
```

Or pull them from the msm8916-mainline alsa-ucm-conf repo:

```bash
git clone https://github.com/msm8916-mainline/alsa-ucm-conf.git
# Copy the samsung-a2015, platforms/msm8916, codecs/msm8916-wcd directories
# into /usr/share/alsa/ucm2/
```

### Verify audio

```bash
cat /proc/asound/cards
# 0 [samsunga2015]: samsung-a2015

alsaucm -c 0 set _verb HiFi
speaker-test -D hw:0,2 -c 1 -t wav
```

---

## Part 12 — Sandboxing (the reason for switching from ALARM)

Trixie ships stable, well-tested package versions. The pmaports kernel config already enables the kernel features required for sandboxing:

```
CONFIG_USER_NS=y
CONFIG_SECCOMP=y
CONFIG_SECCOMP_FILTER=y
CONFIG_CGROUPS=y
```

If you still hit bubblewrap or Flatpak sandbox errors, check:

```bash
cat /proc/sys/kernel/unprivileged_userns_clone
# Must be 1
```

If it is 0:

```bash
echo "kernel.unprivileged_userns_clone=1" | \
  sudo tee /etc/sysctl.d/99-namespaces.conf
sudo sysctl -p /etc/sysctl.d/99-namespaces.conf
```

---

## Part 13 — Touchscreen

Works in default portrait orientation with no config. For display rotation:

```bash
sudo tee /etc/X11/xorg.conf.d/99-display.conf <<EOF
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
EOF
```

---

## Part 14 — Screen Brightness

```bash
echo 100 | sudo tee /sys/class/backlight/1a98000.dsi.0/brightness
```

Allow your user to set it without sudo:

```bash
sudo tee /etc/udev/rules.d/90-backlight.rules <<EOF
ACTION=="add", SUBSYSTEM=="backlight", \
  RUN+="/bin/chmod a+w /sys/class/backlight/%k/brightness"
EOF
```

---

## Troubleshooting

### `debootstrap` second stage fails with missing library errors

The first stage did not complete. Wipe and retry, using `http` not `https`:

```bash
sudo rm -rf /mnt/root/*
sudo debootstrap --arch=arm64 --foreign trixie /mnt/root http://deb.debian.org/debian
```

Watch for a clean end: the last line should be `I: Base system installed successfully.`

### `dpkg-reconfigure` not found inside chroot

Use the full path:

```bash
/usr/sbin/dpkg-reconfigure locales
```

### apt fails with certificate errors inside chroot

Set the date first, then use http until `ca-certificates` is installed:

```bash
date -s "2026-06-05 14:00:00"
sed -i 's/https:/http:/g' /etc/apt/sources.list
apt update
apt install -y ca-certificates
sed -i 's/http:/https:/g' /etc/apt/sources.list
apt update
```

### WiFi / audio / remoteproc not working

All identical to the Arch guide troubleshooting — these are hardware-level issues, not distro-level. Refer to the Arch guide troubleshooting section.

### `rmtfs` build fails: `libqrtr-glib not found`

```bash
sudo apt install -y libqrtr-glib-dev
```

If not in trixie repos, build it first:

```bash
git clone https://gitlab.freedesktop.org/mobile-broadband/libqrtr-glib.git
cd libqrtr-glib
meson setup build
ninja -C build
sudo ninja -C build install
sudo ldconfig
```

Then rebuild rmtfs.

### Kernel updates

The kernel is not managed by apt — update it manually by repeating Part 7 whenever you want a new version. This is intentional: the kernel only changes when you decide it does.

---

## Boot Sequence

```
Power on
  → Samsung/Qualcomm bootloader
  → lk2nd
  → Linux 7.1-msm8916+ starts
  → systemd
      → msm-firmware-loader.service
            mounts apnhlos, modem, persist from internal eMMC (read-only)
            symlinks all blobs into /run/msm-firmware-loader/target/
            sets firmware search path
      → remoteproc1 (WCNSS → wlan0)
      → remoteproc0 (modem → audio DSP path)
      → rmtfs.service
      → NetworkManager (WiFi connects)
      → lightdm (Xorg)
            user logs in
            → icewm + onboard
```

---

## Resources

- [msm8916-mainline kernel](https://github.com/msm8916-mainline/linux)
- [lk2nd](https://github.com/msm8916-mainline/lk2nd)
- [pmaports — msm8916 kernel config](https://gitlab.postmarketos.org/postmarketOS/pmaports/-/tree/master/device/testing/linux-postmarketos-qcom-msm8916)
- [rmtfs](https://github.com/andersson/rmtfs)
- [msm8916-mainline alsa-ucm-conf](https://github.com/msm8916-mainline/alsa-ucm-conf)
- [Debian trixie](https://www.debian.org/releases/trixie/)
