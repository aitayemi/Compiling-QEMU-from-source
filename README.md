# Compiling QEMU from Source — Linux and Windows

QEMU (Quick EMUlator) is a free, open-source machine emulator and virtualizer. It emulates hardware (disks, NICs, graphics controllers) and runs the virtual machine process in user space. For most day-to-day use you'll probably want a management layer like `libvirtd` on top of QEMU/KVM rather than driving QEMU directly — but this guide covers compiling QEMU itself from source, to get full control over which features are built in.

**Why build from source?** The command used here to spin up a VM (with specific `-cpu`/`-smbios` flags to defeat anti-VM detection) does not work correctly against the prebuilt Windows QEMU binary — the VM comes up in a `paused` state. Building from source avoids that.

**Goal:** create a VM where the guest OS believes it's running on physical hardware rather than inside a VM. This is useful for bypassing "Anti-VM" detection used by some games and applications that refuse to run (or misbehave) when they detect a hypervisor.

**Physical environment used throughout this guide:** a Dell laptop running Rocky Linux 9, with an Ubuntu VM used as the build/compiler machine.

---

## Table of Contents

- [1. Host Setup: libvirtd on Rocky/CentOS Stream](#1-host-setup-libvirtd-on-rockycentos-stream)
- [2. Cross-Compiling QEMU for Windows (on an Ubuntu VM)](#2-cross-compiling-qemu-for-windows-on-an-ubuntu-vm)
- [3. Compiling QEMU for Linux (Rocky/RHEL 9.8)](#3-compiling-qemu-for-linux-rockyrhel-98)
- [4. Compiling QEMU Natively on Ubuntu](#4-compiling-qemu-natively-on-ubuntu)
- [5. Camera Passthrough Setup (Host)](#5-camera-passthrough-setup-host)
- [6. Sample VM Launch Commands](#6-sample-vm-launch-commands)
  - [6.1 Linux host (KVM) — Windows 10 guest, with audio + camera](#61-linux-host-kvm--windows-10-guest-with-audio--camera)
  - [6.2 Windows host (WHPX) — Windows 10 guest, with audio + camera](#62-windows-host-whpx--windows-10-guest-with-audio--camera)
- [7. Troubleshooting Notes](#7-troubleshooting-notes)
- [Appendix A: Microphone Not Working in the VM](#appendix-a-microphone-not-working-in-the-vm)

---

## 1. Host Setup: libvirtd on Rocky/CentOS Stream

Set this up on the physical host so you can create a VM to use as your compiler machine.

```bash
# 1. Check hardware virtualization support
grep -e 'vmx' -e 'svm' /proc/cpuinfo

# 2. Install virtualization packages
sudo dnf install -y qemu-kvm libvirt virt-manager virt-install virt-viewer virt-top bridge-utils libguestfs-tools

# 3. Start and enable libvirtd
sudo systemctl enable --now libvirtd
sudo systemctl status libvirtd

# 4. Configure user permissions
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER
newgrp libvirt

# 5. Start the default network
sudo virsh net-start default
sudo virsh net-autostart default

# 6. Allow an ordinary (non-root) user to create a VM with working networking
ip a                                   # find your bridge interface, e.g. virbr0
sudo mkdir -p /etc/qemu
echo "allow virbr0" | sudo tee /etc/qemu/bridge.conf
sudo chmod 644 /etc/qemu/bridge.conf
sudo chmod u+s /usr/libexec/qemu-bridge-helper
```

---

## 2. Cross-Compiling QEMU for Windows (on an Ubuntu VM)

### 2.1 Create the Ubuntu build VM

```bash
mkdir /VMs && cd /VMs
wget https://cloud-images.ubuntu.com/minimal/releases/noble/release/ubuntu-24.04-minimal-cloudimg-amd64.img -O compiler.qcow2

# The base image is only ~3.5G — grow it
qemu-img resize compiler.qcow2 30g

cat >> user-data << 'EOF'
#cloud-config
user: ubuntu
password: xxxxx123!
chpasswd: { expire: False }
ssh_pwauth: True
# Automatically updates the repositories and installs ping on first boot
packages:
  - iputils-ping
EOF

sudo usermod -aG libvirt $USER
newgrp libvirt

sudo virt-install --name compiler --memory 8192 --vcpus 4 \
  --disk compiler.qcow2,format=qcow2 --os-variant ubuntu24.04 \
  --import --graphics none --network default \
  --cloud-init user-data=user-data
```

Log in with the credentials from `user-data`, then confirm the OS version:

```bash
cat /etc/os-release | grep VERSION=
# VERSION="24.04.5 LTS (Noble Numbat)"
```

Optional shell quality-of-life tweaks:

```bash
echo "export TERM=vt220" >> ~/.bashrc
echo "set enable-bracketed-paste off" >> ~/.bashrc
echo 'export PS1="[\u@\h \W]\$"' >> ~/.bashrc
echo "PS2='>'" >> ~/.bashrc
source ~/.bashrc
```

### 2.2 Install build dependencies

```bash
sudo apt update -y && sudo apt upgrade -y
sudo apt install -y mingw-w64 software-properties-common
sudo add-apt-repository -c universe -y
sudo apt update -y && sudo apt upgrade -y

DEBIAN_FRONTEND=noninteractive sudo -E apt install -y \
  autoconf automake autopoint bash bison bzip2 flex gettext git g++ gperf \
  intltool libc6-dev-i386 libgdk-pixbuf2.0-dev libltdl-dev libgl-dev \
  libpcre3-dev libssl-dev libtool-bin libxml-parser-perl lzip make openssl \
  p7zip-full patch perl pkg-config python3 python3-mako python3-pkg-resources \
  ruby sed unzip wget xz-utils g++-multilib python-is-python3 python3-venv \
  vim meson libusb-1.0-0-dev libibverbs-dev librdmacm-dev libcacard-dev \
  libusbredirparser-dev
```

> If prompted for keyboard layout: enter `35` (English US) or `34` (English UK) for country, then `1` (English US) for keyboard layout.

### 2.3 Build the MinGW toolchain (MXE) and static dependencies

```bash
cd
git clone https://github.com/mxe/mxe.git
cd mxe/
make MXE_TARGETS='x86_64-w64-mingw32.static' glib gtk3 pixman sdl2 openssl zlib jpeg opus orc libusb1 lz4
make MXE_TARGETS='x86_64-w64-mingw32.static' lz4

echo 'export PATH="/home/ubuntu/mxe/usr/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### 2.4 Build libslirp (static, cross-compiled)

```bash
cd
git clone https://gitlab.freedesktop.org/slirp/libslirp.git
cd libslirp

cat > mxe-cross.txt << 'EOF'
[binaries]
c = 'x86_64-w64-mingw32.static-gcc'
ar = 'x86_64-w64-mingw32.static-ar'
strip = 'x86_64-w64-mingw32.static-strip'
pkg-config = 'x86_64-w64-mingw32.static-pkg-config'
windres = 'x86_64-w64-mingw32.static-windres'

[host_machine]
system = 'windows'
cpu_family = 'x86_64'
cpu = 'x86_64'
endian = 'little'
EOF

meson setup build --cross-file mxe-cross.txt \
  --prefix=/home/ubuntu/mxe/usr/x86_64-w64-mingw32.static --default-library=static
ninja -C build install
x86_64-w64-mingw32.static-pkg-config --modversion slirp
```

### 2.5 Build spice-protocol

```bash
cd ~
git clone https://gitlab.freedesktop.org/spice/spice-protocol.git
cd spice-protocol
meson setup build --prefix=/tmp/spice-protocol-install
ninja -C build install

cp -r /tmp/spice-protocol-install/include/spice-1 ~/mxe/usr/x86_64-w64-mingw32.static/include/
cp /tmp/spice-protocol-install/share/pkgconfig/spice-protocol.pc ~/mxe/usr/x86_64-w64-mingw32.static/lib/pkgconfig/
sed -i "s|/tmp/spice-protocol-install|/home/ubuntu/mxe/usr/x86_64-w64-mingw32.static|g" \
  ~/mxe/usr/x86_64-w64-mingw32.static/lib/pkgconfig/spice-protocol.pc
x86_64-w64-mingw32.static-pkg-config --modversion spice-protocol
```

### 2.6 Build orc

```bash
cd ~
git clone https://gitlab.freedesktop.org/gstreamer/orc.git
cd orc
meson setup build --cross-file ~/libslirp/mxe-cross.txt \
  --prefix=/home/ubuntu/mxe/usr/x86_64-w64-mingw32.static --default-library=static
ninja -C build install
x86_64-w64-mingw32.static-pkg-config --modversion orc-0.4
```

Sanity-check every dependency before tackling spice-server, since a missed one there is a far more expensive failure to debug:

```bash
for pkg in openssl zlib libjpeg opus orc-0.4 glib-2.0 pixman-1 liblz4; do
  echo -n "$pkg: "
  x86_64-w64-mingw32.static-pkg-config --modversion "$pkg" 2>&1
done
```

### 2.7 Build spice-server

```bash
cd ~
sed -i "/^c = /a cpp = 'x86_64-w64-mingw32.static-g++'" ~/libslirp/mxe-cross.txt
cat ~/libslirp/mxe-cross.txt

# The build might fail partway — keep going
wget https://www.spice-space.org/download/releases/spice-server/spice-0.15.2.tar.bz2
tar xjf spice-0.15.2.tar.bz2
cd ~/spice-0.15.2
rm -rf build

meson setup build --cross-file ~/libslirp/mxe-cross.txt \
  --prefix=/home/ubuntu/mxe/usr/x86_64-w64-mingw32.static --default-library=static \
  -Dgstreamer=no -Dlz4=true -Dsasl=false -Dsmartcard=disabled -Dmanual=false \
  -Dstatistics=false -Dopus=enabled -Dspice-common:tests=false -Dtests=false \
  -Dc_link_args=-Wl,--allow-multiple-definition

ninja -C build install
x86_64-w64-mingw32.static-pkg-config --modversion spice-server
```

### 2.8 Build QEMU itself (cross-compiled for Windows)

```bash
cd
git clone https://git.qemu.org/git/qemu.git --depth 1
cd qemu
git submodule init
git submodule update --recursive

rm -rf build-win64
mkdir build-win64 && cd ~/qemu/build-win64

../configure \
  --cross-prefix=x86_64-w64-mingw32.static- \
  --target-list=x86_64-softmmu \
  --extra-cflags="-D__USE_MINGW_ANSI_STDIO=1 -Wno-error=format -Wno-error=suggest-attribute=format -Wno-error=format-extra-args -DLIBSLIRP_STATIC" \
  --extra-ldflags="-lstdc++" \
  --enable-slirp --enable-spice --enable-gtk --enable-vnc --enable-whpx \
  --audio-drv-list=dsound,sdl --enable-libusb --enable-sdl \
  --prefix=~/qemu-win

make -j$(nproc)
make install
```

### 2.9 Package the Windows build

```bash
cd ~/qemu-win
find ./ -iname '*.exe' -exec cp {} ~/qemu-win/ \;
mkdir ~/qemu-win/share
cp ../pc-bios/*.bin ../pc-bios/*.rom ./pc-bios/*x86_64*.fd ~/qemu-win/share/
cd ~

# Optional: drop support for architectures you won't emulate, to shrink the package
cd ~/qemu-win/
find ./ -iname '*aarch64*' -exec rm -f {} \;
find ./ -iname '*riscv*' -exec rm -f {} \;
find ./ -iname '*loongarch64*' -exec rm -f {} \;

# Archive for transfer to a Windows host
cd
tar czf ~/qemu-win.tgz ./qemu-win
```

Transfer `qemu-win.tgz` to your Windows host, extract it into a folder such as `C:\qemu-win\`, and run `qemu-system-x86_64.exe` from there.

---

## 3. Compiling QEMU for Linux (Rocky/RHEL 9.8)

### 3.1 Create the Rocky build VM

```bash
mkdir /VMs && cd /VMs
wget https://dl.rockylinux.org/pub/rocky/9/images/x86_64/Rocky-9-GenericCloud-Base.latest.x86_64.qcow2 -O compiler.qcow2
qemu-img resize compiler.qcow2 30G

cat > user-data << 'EOF'
#cloud-config
user: vmuser
password: xxxxx123!
chpasswd: { expire: False }
ssh_pwauth: True
# Automatically updates the repositories and installs python3.14 on first boot
packages:
  - python3.14
EOF

sudo virt-install --name compiler --memory 8192 --vcpus 4 \
  --disk compiler.qcow2,format=qcow2 --os-variant rhel9.8 \
  --import --graphics none --network default \
  --cloud-init user-data=user-data
```

Log in with the credentials from `user-data`, then verify:

```bash
cat /etc/os-release | grep VERSION=
# VERSION="9.8 (Blue Onyx)"

df -h /
sudo dnf update -yq
```

### 3.2 Install build dependencies

QEMU requires Python 3.12+; Rocky 9's default is 3.9, so 3.14 is installed alongside it.

```bash
sudo dnf install -y epel-release
sudo dnf config-manager --set-enabled crb
sudo dnf install -y 'dnf-command(copr)'
sudo dnf copr enable -y ligenix/enterprise-qemu-spice

sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y \
  glib2-devel pixman-devel zlib-devel ninja-build meson python3-devel \
  python3-pip SDL2-devel libusb1-devel libusbx-devel usbredir-devel \
  rdma-core-devel libibverbs-devel libslirp-devel gnutls-devel nettle-devel \
  libcap-ng-devel libattr-devel pciutils-devel bzip2-devel snappy-devel \
  libcurl-devel numactl-devel libaio-devel python3.14 python3.14-pip python3.14-pip \
  python3.14-setuptools python3.14-devel spice-server-devel gtk3-devel alsa-lib-devel pulseaudio-libs-devel wget
```

Switch the active `python3` to 3.14 (needed since some build utilities require 3.12+):

```bash
sudo alternatives --install /usr/bin/python3 python3 /usr/bin/python3.14 1
sudo alternatives --set python3 /usr/bin/python3.14
python -V
```

> **Caution:** redirecting system-wide `python3` away from the OS-provided 3.9 can break tools that depend on it (e.g. `dnf`). Consider scoping this to a venv instead if you're doing this outside a disposable build VM.

### 3.3 Build QEMU

```bash
git clone https://git.qemu.org/git/qemu.git --depth 1
cd qemu
git submodule init
git submodule update --recursive

rm -rf build
mkdir build && cd build

../configure \
  --target-list=x86_64-softmmu \
  --extra-cflags="-D__USE_MINGW_ANSI_STDIO=1 -Wno-error=format -Wno-error=suggest-attribute=format -Wno-error=format-extra-args -DLIBSLIRP_STATIC" \
  --extra-ldflags="-lstdc++" \
  --enable-slirp --enable-spice --enable-gtk --enable-vnc --enable-alsa --enable-pa \
  --prefix=/opt/qemu

make -j$(nproc)
sudo make install

cd /
tar czf ~/qemu-linux.tgz ./opt/qemu
```

- NOTE: Copy `qemu-linux.tgz` to another Linux host to deploy the build.
- NOTE: if you want to compile QEMU for all supported platforms (POWER/PPC, SPARC, etc), remove (–target-list=x86_64-softmmu) and add --disable-werror to the configure command.
---

## 4. Compiling QEMU Natively on Ubuntu

```bash
mkdir /VMs && cd /VMs
wget https://cloud-images.ubuntu.com/minimal/releases/noble/release/ubuntu-24.04-minimal-cloudimg-amd64.img -O compiler.qcow2
qemu-img resize compiler.qcow2 30g

cat >> user-data << 'EOF'
#cloud-config
user: ubuntu
password: xxxxx123!
chpasswd: { expire: False }
ssh_pwauth: True
packages:
  - iputils-ping
EOF

sudo usermod -aG libvirt $USER
newgrp libvirt

sudo virt-install --name compiler --memory 8192 --vcpus 4 \
  --disk compiler.qcow2,format=qcow2 --os-variant ubuntu24.04 \
  --import --graphics none --network default \
  --cloud-init user-data=user-data
```

Log in, then install build dependencies:

```bash
sudo apt update -y && sudo apt upgrade -y
sudo apt install -y mingw-w64 software-properties-common
sudo add-apt-repository -c universe -y
sudo apt update -y && sudo apt upgrade -y

DEBIAN_FRONTEND=noninteractive sudo -E apt install -y \
  autoconf automake autopoint bash bison bzip2 flex gettext git g++ gperf \
  intltool libc6-dev-i386 libgdk-pixbuf2.0-dev libltdl-dev libgl-dev \
  libpcre3-dev libssl-dev libtool-bin libxml-parser-perl lzip make openssl \
  p7zip-full patch perl pkg-config python3 python3-mako python3-pkg-resources \
  ruby sed unzip wget xz-utils g++-multilib python-is-python3 python3-venv \
  vim meson libusb-1.0-0-dev libibverbs-dev librdmacm-dev libcacard-dev \
  libusbredirparser-dev

sudo apt install -y \
  build-essential ninja-build meson pkg-config libglib2.0-dev libpixman-1-dev \
  libgtk-3-dev libvncserver-dev libslirp-dev libspice-server-dev \
  libspice-protocol-dev libsdl2-dev libssl-dev zlib1g-dev
```

Build:

```bash
rm -rf qemu
git clone https://git.qemu.org/git/qemu.git --depth 1
cd qemu
git submodule init
git submodule update --recursive

rm -rf build
mkdir build && cd build

../configure \
  --target-list=x86_64-softmmu \
  --extra-cflags="-D__USE_MINGW_ANSI_STDIO=1 -Wno-error=format -Wno-error=suggest-attribute=format -Wno-error=format-extra-args -DLIBSLIRP_STATIC" \
  --extra-ldflags="-lstdc++" \
  --enable-slirp --enable-spice --enable-gtk --enable-vnc \
  --prefix=/opt/qemu

make -j$(nproc)
sudo make install

cd /
tar czf ~/qemu-linux.tgz ./opt/qemu
```

- NOTE: Copy `qemu-linux.tgz` to another Linux host as needed.
- NOTE: If you want to compile QEMU for all supported platforms (POWER/PPC, SPARC, etc), remove (–target-list=x86_64-softmmu) and add --disable-werror to the configure command. 
---

## 5. Camera Passthrough Setup (Host)

Done on the physical Linux host (Rocky/CentOS Stream or Ubuntu) before launching a VM that needs camera access.

```bash
# Get the vendor/product ID of the camera
lsusb | grep -i cam
# Bus 003 Device 002: ID 1bcf:2ba9 Sunplus Innovation Technology Inc. Integrated_Webcam_FHD

lsusb -d 1bcf:2ba9

# Create a udev rule granting access by vendor/product ID
sudo tee /etc/udev/rules.d/99-qemu-usb.rules <<'EOF'
SUBSYSTEM=="usb", ATTR{idVendor}=="1bcf", ATTR{idProduct}=="2ba9", MODE="0666", GROUP="kvm"
EOF

sudo udevadm control --reload-rules
sudo udevadm trigger

ls -l /dev/video*
ls -l /dev/bus/usb/003/002
```

---

## 6. Sample VM Launch Commands

### 6.1 Linux host (KVM) — Windows 10 guest, with audio + camera

```bash
cd /opt/qemu/bin

./qemu-system-x86_64 -accel kvm \
  -cpu host,hv_relaxed,hv_vapic,hv_time \
  -machine q35 \
  -L /opt/qemu/share/qemu \
  -smbios type=0,vendor="American Megatrends International,, LLC.",version="F10" \
  -smbios type=1,manufacturer="ASUSTeK COMPUTER INC.",product="PRIME Z790-A WIFI",version="Rev 1.xx",serial="1234567890" \
  -smbios type=2,manufacturer="ASUSTeK COMPUTER INC.",product="PRIME Z790-A WIFI",version="Rev 1.xx",serial="1234567890" \
  -smbios type=3,manufacturer="ASUSTeK COMPUTER INC.",version="Rev 1.xx",serial="1234567890" \
  -netdev user,id=net0 \
  -device e1000e,netdev=net0,mac=00:1A:2B:3C:4D:5E \
  -drive file=~/mypc.qcow2,if=none,id=drive-sata0,cache=writeback,format=raw \
  -device ide-hd,drive=drive-sata0,bus=ide.0,serial="WDC-WD10EZEX-00BN7A0",model="Western Digital" \
  -drive file=/VMs/Windows1022H2.iso,if=none,id=drive-cd0,media=cdrom \
  -device ide-cd,drive=drive-cd0,bus=ide.1 \
  -boot order=c,once=d,menu=on \
  -m 16G -smp 4 -vga qxl \
  -audiodev sdl,id=snd0 -device ich9-intel-hda -device hda-micro,audiodev=snd0 \
  -device nec-usb-xhci,id=usbctrl \
  -device usb-host,bus=usbctrl.0,vendorid=0x1BCF,productid=0x2BA9,suppress-remote-wake=on,pipeline=off
```

- Drop `once=d,` and the CD-ROM `-drive`/`-device` pair once the OS is already installed and you no longer need the ISO.
- `hda-micro` provides a real microphone input; `hda-duplex` only gives a line-in, which most guest OSes won't surface as a mic.
- If the guest sees a microphone but records silence, see [Appendix A](#appendix-a-microphone-not-working-in-the-vm) — the cause is often a muted capture path on the physical host.
- Alternative board identity (Dell instead of ASUS):
  ```
  -smbios type=1,manufacturer="Dell Inc.",product="OptiPlex 7090",version="1.0",serial="ABC123XYZ" \
  -smbios type=2,manufacturer="Dell Inc.",product="0XYZ12",version="A01" \
  -smbios type=3,manufacturer="Dell Inc."
  ```

### 6.2 Windows host (WHPX) — Windows 10 guest, with audio + camera

```cmd
cd C:\qemu-win
qemu-img.exe create -f qcow2 mypc.qcow2 35g

qemu-system-x86_64.exe -accel whpx -cpu host,hv_relaxed,hv_vapic,hv_time -machine q35 -L C:\qemu-win\share ^
  -smbios type=0,vendor="American Megatrends International,, LLC.",version="F10" ^
  -smbios type=1,manufacturer="ASUSTeK COMPUTER INC.",product="PRIME Z790-A WIFI",version="Rev 1.xx",serial="1234567890" ^
  -smbios type=2,manufacturer="ASUSTeK COMPUTER INC.",product="PRIME Z790-A WIFI",version="Rev 1.xx",serial="1234567890" ^
  -smbios type=3,manufacturer="ASUSTeK COMPUTER INC.",version="Rev 1.xx",serial="1234567890" ^
  -netdev user,id=net0 -device e1000e,netdev=net0,mac=00:1A:2B:3C:4D:5E ^
  -drive file=C:\qemu-win\mypc.qcow2,if=none,id=drive-sata0,cache=writeback ^
  -device ide-hd,drive=drive-sata0,bus=ide.0,serial="WDC-WD10EZEX-00BN7A0",model="Western Digital" ^
  -boot order=c,once=d,menu=on -m 16G -smp 4 -vga qxl ^
  -audiodev dsound,id=snd0 -device ich9-intel-hda -device hda-micro,audiodev=snd0 ^
  -device nec-usb-xhci,id=usbctrl ^
  -device usb-host,bus=usbctrl.0,vendorid=0x1BCF,productid=0x2BA9
```

- Must be run from an **elevated** (Run as Administrator) command prompt for VM networking to work.
- Audio (speaker and microphone) works; camera passthrough via `usb-host` is **not** detected on Windows hosts in testing.
- If an installer-initiated restart hangs with `qemu: WHPX: Unexpected VP exit code 4`, close QEMU and relaunch — the install resumes from where it left off.
- Add the CD-ROM lines back (`-drive file=...,media=cdrom -device ide-cd,...`) and keep `once=d,` while doing a fresh install from ISO.

---

## 7. Troubleshooting Notes

**Windows 10 Activation fails ("Something went wrong — But you can try again") during OOBE**
Usually caused by Windows not recognizing the emulated NIC during setup.

1. On the error screen, press **Shift + F10** (or **Fn + Shift + F10**) to open a Command Prompt.
2. Click the Command Prompt window to make sure it's focused.
3. Run `start ms-cxh:localonly` and press Enter.
4. This opens the local-account setup flow so you can create an offline username instead of signing in with a Microsoft account.

**Installer hangs on a black screen after a restart (KVM/Linux host)**
Just terminate the QEMU process (or close the window) and relaunch it — the installation resumes from where it left off.

**Remote display note**
When connected to the physical Linux host over SSH (e.g. via MobaXterm from a Windows client), the QEMU GUI launches on the local X server through X11 forwarding.

**Anti-VM detection**
With the SMBIOS spoofing shown above, even applications that won't run in s VM runs as they can't tell they are being launched in a VM.

---

## Appendix A: Microphone Not Working in the VM

**Symptom:** the microphone shows up in the guest (e.g. in Windows Sound settings) but records nothing. QEMU reports no error — there is just no sound.

**Cause found here:** the microphone was also silent on the physical host, because the capture path was muted/unboosted in the host's ALSA mixer. QEMU can only pass through what the host itself can capture, so always test the mic on the host first.

> **Remote sessions:** the VM uses the audio hardware of the machine QEMU runs on. If you connect over SSH/MobaXterm, the guest gets the *host's* microphone, not the one on the PC you're sitting at. Playback from `aplay` also comes out of the host's speakers, so you can't judge a test recording by ear when remote — use the level meter in [A.3](#a3-re-test-with-the-level-meter) instead.

### A.1 List the real mixer control names

Control names vary by codec, so don't assume — list what your card actually exposes:

```bash
$ amixer -c 0 scontrols
Simple mixer control 'Master',0
Simple mixer control 'Headphone',0
Simple mixer control 'Headphone Mic',0
Simple mixer control 'Headphone Mic Boost',0
Simple mixer control 'Speaker',0
Simple mixer control 'PCM',0
Simple mixer control 'IEC958',0
Simple mixer control 'IEC958',1
Simple mixer control 'IEC958',2
Simple mixer control 'IEC958',3
Simple mixer control 'Capture',0
Simple mixer control 'Auto-Mute Mode',0
Simple mixer control 'Headset Mic',0
Simple mixer control 'Headset Mic Boost',0
Simple mixer control 'Internal Mic',0
Simple mixer control 'Internal Mic Boost',0
```

(Output above is from a Dell laptop with a Realtek ALC3204 codec. Use `arecord -l` to confirm which card/device number your capture hardware is.)

### A.2 Unmute and boost the mic

Substitute the real control names from A.1. On this laptop the built-in mic is **Internal Mic**:

```bash
amixer -c 0 sset 'Capture' cap unmute
amixer -c 0 sset 'Capture' 80%
amixer -c 0 sset 'Internal Mic Boost' 2
```

Notes:

- `Front Mic Boost` and `Input Source` do not exist on this codec. If your `scontrols` list shows them, set them too (e.g. `amixer -c 0 sset 'Input Source' 'Internal Mic'`); otherwise skip them.
- If you plug in a headset instead, use `Headset Mic` / `Headset Mic Boost` (or `Headphone Mic` / `Headphone Mic Boost`, depending on the jack).
- `alsamixer -c 0` then **F4** shows the capture view graphically; channels marked `MM` are muted and can be toggled with **Space**.

### A.3 Re-test with the level meter

```bash
arecord -D hw:0,0 -f cd -vv /dev/null
```

The percentage meter should go up and down as you talk or tap near the mic. If it stays at `00%`, there is still a problem on the host — keep working through A.2 (and check that a mic is plugged in if your hardware has no built-in one) before touching QEMU. Press **Ctrl+C** to stop.

### A.4 Point QEMU at ALSA directly (optional)

If the mic works on the host but is still silent in the VM, bypass SDL and the sound server by using the ALSA backend for the device you tested (`hw:0,0`):

```
-audiodev alsa,id=snd0,in.dev=hw:0,0,out.dev=default
```

This replaces `-audiodev sdl,id=snd0` in the launch command from [6.1](#61-linux-host-kvm--windows-10-guest-with-audio--camera). If QEMU reports the device as busy, PipeWire/PulseAudio is holding it; use `-audiodev pipewire,id=snd0` or `-audiodev pa,id=snd0` instead (run `./qemu-system-x86_64 -audiodev help` to see which backends your build supports).

### A.5 Make the settings survive a reboot

`amixer` changes only affect the running system. Save them so they are restored at boot:

```bash
sudo dnf install -y alsa-utils
sudo alsactl store
```

To verify:

```bash
grep -A6 "Internal Mic Boost" /var/lib/alsa/asound.state
systemctl status alsa-restore.service
```
