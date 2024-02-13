Contents
========

* [Summary](#summary)
* [GRUB](#grub)
* [persistent-evdev](#persistent-evdev)
* [udev](#udev)
* [Looking Glass](#looking-glass)
* [Sync screen blanking](#sync-screen-blanking)
* [PipeWire](#pipewire-and-x11-forwarding)

Summary
=======

OVMF on NVIDIA laptop with Arch host, Windows 10 guest, pcie passthrough, Looking Glass for laptop monitor, synchronized screen idle blanking, bluetooth device passthrough and hotplugging with evdev, GRUB boot entries to switch between Windows having the GPU and Arch retaining control of it

Leave a message on my ArchWiki talk page if you have any questions about my VFIO setup. https://wiki.archlinux.org/title/User_talk:Alydev

Hardware:

* **Laptop**: Acer Nitro 5 AN515-57
* **CPU**: i5-11400H @ 2.70GHz
* **GPU**: GeForce RTX 3050 Mobile
* **RAM**: 24gb generic SODIMM DDR4

Configuration:

* **Kernel**: 6.7.4-arch1-1 [GRUB](#grub)
* Using **libvirt/QEMU**: [/etc/libvirt/qemu/win10.xml](/win10.xml)
* Using **persistent-evdev** for hotplugging and bluetooth passthrough: https://github.com/aiberia/persistent-evdev
  * [persistent-evdev config](#persistent-evdev)
  * [udev rules](#udev)
* **Looking Glass** on Windows for laptop monitor: [Looking Glass](#looking-glass)
* **DPMS trigger over SSH**: [Sync screen blanking](#sync-screen-blanking)

GRUB
====

`/etc/default/grub`
```bash
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet modprobe.blacklist=vfio-pci"
```

`/etc/grub.d/40_custom`
```grub
menuentry "Arch Linux - PCI Passthrough" {
        load_video
        set gfxpayload=keep
        insmod gzio
        insmod part_gpt
        insmod ext2
        search --no-floppy --fs-uuid --set=root d8dfdf74-8c7d-4d1e-a012-79abc959a420
        echo    'Loading Linux linux ...'
        linux   /vmlinuz-linux root=UUID=48247929-96dd-45d0-853e-070a96bf73ce rw  loglevel=3 quiet rd.driver.pre=vfio-pci intel_iommu=on vfio-pci.ids=10de:25a2,10de:2291 modprobe.blacklist=nvidia
        echo    'Loading initial ramdisk ...'
        initrd  /initramfs-linux.img
}
```

persistent-evdev
================

`/opt/persistent-evdev/etc/config`
```json
{
    "cache": "/opt/persistent-evdev/cache",
    "devices": {
        "persist-mouse0": "/dev/input/btmouse",
        "persist-mouse1": "/dev/input/by-id/usb-Razer_Razer_Naga_Trinity_00000000001A-event-mouse",
        "persist-keyboard0": "/dev/input/btkeyboard",
        "persist-keyboard1": "/dev/input/by-id/usb-Logitech_G512_Carbon_GX_Blue_065B38693437-event-kbd"
    }
}
```

udev
====

`/etc/udev/rules.d/60-persistent-input-uinput.rules`
```udev
KERNEL=="event*", ATTRS{phys}=="py-evdev-uinput", ATTRS{name}=="?*", SYMLINK+="input/by-id/uinput-$attr{name}"
```

`/etc/udev/rules.d/80-bluetooth.rules`
```udev
KERNEL=="event[0-9]|event[0-9][0-9]", SUBSYSTEM=="input", ATTRS{name}=="Keyboard K480 Keyboard", ACTION=="add", SYMLINK+="input/btkeyboard", OWNER="aly", GROUP="libvirt-qemu"
KERNEL=="event[0-9]|event[0-9][0-9]", SUBSYSTEM=="input", ATTRS{name}=="LOGI M240 Mouse", ACTION=="add", SYMLINK+="input/btmouse", OWNER="aly", GROUP="libvirt-qemu"
```

Looking Glass
=============

Arch
----

`/etc/tmpfiles.d/10-looking-glass.conf`
```
f /dev/shm/looking-glass 0660 USER kvm -
```

`$HOME/.xinitrc`
```bash
#!/bin/sh
fluxbox &
sleep 1
looking-glass-client -F -s
```

Windows
-------

https://github.com/ge9/IddSampleDriver, configured to create 1080p display.

Looking Glass needs to select monitor #2:

`C:\Program Files\Looking Glass (host)\looking-glass-host.ini`
```ini
[dxgi]
adapter=0
output=2
```

Sync screen blanking
====================

Bit of a hack. Press enter while screens are off to pass through pause and turn the second screen back on.

`%USERPROFILE%\Documents\screenoff.ps1`
```ps1
ssh 192.168.122.1 .local/bin/screenoff
(Add-Type '[DllImport("user32.dll")]public static extern int SendMessage(int hWnd, int hMsg, int wParam, int lParam);' -Name a -Pas)::SendMessage(-1,0x0112,0xF170,2)
pause
ssh 192.168.122.1 .local/bin/screenon
```

`$HOME/.local/bin/screenoff`
```bash
#!/bin/sh
PS="$(ps aux | grep Xorg | grep -v grep)"
export DISPLAY=$(awk '{print $(NF-4)}' <<<"$PS")
export XAUTHORITY=$(awk '{print $NF}' <<<"$PS")
xset dpms force off
```

`$HOME/.local/bin/screenon`
```bash
#!/bin/sh
PS="$(ps aux | grep Xorg | grep -v grep)"
export DISPLAY=$(awk '{print $(NF-4)}' <<<"$PS")
export XAUTHORITY=$(awk '{print $NF}' <<<"$PS")
xset dpms force on
```

PipeWire and X11 Forwarding
========

I didn't want to change Pipewire to running as a system daemon. This is my workaround. After booting with my [PCI Passthrough boot option](#grub), I press ctrl+alt+f3, this takes me to the autologin shell, in which bashrc detects that vfio is loaded and starts the win10 VM if not already running. Obvious caveat is the security issue of autologin. But anyone with physical access to the VT already has access what's the difference.

`/etc/systemd/system/getty@tty3.service.d/autologin.conf`
```systemd
[Service]
ExecStart=
ExecStart=-/sbin/agetty -o '-p -f -- \\u' --noclear --autologin aly %I $TERM
```

`$HOME/.bashrc`
```bash
export LIBVIRT_DEFAULT_URI=qemu:///system
[ "$(tty)" == "/dev/tty3" ] && [ -e "/sys/bus/pci/drivers/vfio-pci/0000:01:00.0" ] && [ "$(virsh list | grep win10 | awk '{print $NF}')" != "running" ] && virsh start win10
if [ -n "$SSH_CONNECTION" ] && [ -n "$DISPLAY" ]; then
        export SSH_GUEST=$(awk '{print $1}' <<< "$SSH_CONNECTION")
        export DISPLAY=$SSH_GUEST:0
fi
```
