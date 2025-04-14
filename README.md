# 3D accelerated qemu on MacOS

> **Note:** This is a fork of the original `knazarov/homebrew-qemu-virgl` tap. It incorporates patches that fix certain issues (like  missing FreeDOS 1.2 verification image (FD12FLOPPY.zip) and adds improved support for Apple's `vmnet.framework` for improved networking features on macOS.

![ubuntu](https://user-images.githubusercontent.com/6728841/111193747-90da1a00-85cb-11eb-9517-36c1a19c19be.gif)

## What is it for

If you own a Mac (x86 or ARM) and want to have a full Linux desktop for development or testing, you'll find that having a responsive desktop is a nice thing. The graphical acceleration is possible thanks to [the work](https://gist.github.com/akihikodaki/87df4149e7ca87f18dc56807ec5a1bc5) of [Akihiko Odaki](https://github.com/akihikodaki). I've only packaged it into an easily-installable brew repository while the changes are not yet merged into upstream.

Features:

- Support for both ARM and X86 acceleration with Hypervisor.framework (works without root or kernel extensions)
- Support for OpenGL acceleration in the guest (both X11 and Wayland)
- Works on large screens (5k+)
- Dynamically changing guest resolution on window resize
- Properly handle sound output when plugging/unplugging headphones

## Installation

`brew install knazarov/qemu-virgl/qemu-virgl`

Or `brew tap knazarov/qemu-virgl` and then `brew install qemu-virgl`.


## Usage

Qemu has many command line options and emulated devices, so the
sections are specific to your CPU (Intel/M1).

For the best experience, maximize the qemu window when it starts. To
release the mouse, press `Ctrl-Alt-g`.

### Usage - M1 Macs

**Latest release needs virtio-gpu-gl-pci command line option instead of virtio-gpu-pci, otherwise gpu acceleration won't work**

First, create a disk image you'll run your Linux installation from (tune image size as needed):

```sh
qemu-img create hdd.raw 64G
```

Download an ARM based Fedora 35 image:

```sh
curl -LO https://www.mirrorservice.org/sites/dl.fedoraproject.org/pub/fedora/linux/releases/35/Workstation/aarch64/iso/Fedora-Workstation-Live-aarch64-35-1.2.iso
```

Copy the firmware:

```sh
cp $(dirname $(which qemu-img))/../share/qemu/edk2-aarch64-code.fd .
cp $(dirname $(which qemu-img))/../share/qemu/edk2-arm-vars.fd .
```

Install the system from the CD image:

```sh
qemu-system-aarch64 \
         -machine virt,accel=hvf,highmem=off \
         -cpu cortex-a72 -smp 2 -m 4G \
         -device intel-hda -device hda-output \
         -device qemu-xhci \
         -device virtio-gpu-gl-pci \
         -device usb-kbd \
         -device virtio-net-pci,netdev=net \
         -device virtio-mouse-pci \
         -display cocoa,gl=es \
         -netdev user,id=net,ipv6=off \
         -drive "if=pflash,format=raw,file=./edk2-aarch64-code.fd,readonly=on" \
         -drive "if=pflash,format=raw,file=./edk2-arm-vars.fd,discard=on" \
         -drive "if=virtio,format=raw,file=./hdd.raw,discard=on" \
         -cdrom Fedora-Workstation-Live-aarch64-35-1.2.iso \
         -boot d
```

Run the system without the CD image to boot into the primary partition:

```sh
qemu-system-aarch64 \
         -machine virt,accel=hvf,highmem=off \
         -cpu cortex-a72 -smp 2 -m 4G \
         -device intel-hda -device hda-output \
         -device qemu-xhci \
         -device virtio-gpu-gl-pci \
         -device usb-kbd \
         -device virtio-net-pci,netdev=net \
         -device virtio-mouse-pci \
         -display cocoa,gl=es \
         -netdev user,id=net,ipv6=off \
         -drive "if=pflash,format=raw,file=./edk2-aarch64-code.fd,readonly=on" \
         -drive "if=pflash,format=raw,file=./edk2-arm-vars.fd,discard=on" \
         -drive "if=virtio,format=raw,file=./hdd.raw,discard=on"
```


### Usage - Intel Macs

First, create a disk image you'll run your Linux installation from (tune image size as needed):

```sh
qemu-img create hdd.raw 64G
```

Download an x86 based Fedora 35 image:

```sh
curl -LO https://www.mirrorservice.org/sites/dl.fedoraproject.org/pub/fedora/linux/releases/35/Workstation/x86_64/iso/Fedora-Workstation-Live-x86_64-35-1.2.iso
```

Install the system from the CD image:

```sh
qemu-system-x86_64 \
         -machine accel=hvf \
         -cpu Haswell-v4 -smp 2 -m 4G \
         -device intel-hda -device hda-output \
         -device qemu-xhci \
         -device virtio-vga-gl \
         -device usb-kbd \
         -device virtio-net-pci,netdev=net \
         -device virtio-mouse-pci \
         -display cocoa,gl=es \
         -netdev user,id=net,ipv6=off \
         -drive "if=virtio,format=raw,file=hdd.raw,discard=on" \
         -cdrom Fedora-Workstation-Live-x86_64-35-1.2.iso \
         -boot d
```

Run the system without the CD image to boot into the primary partition:

```sh
qemu-system-x86_64 \
         -machine accel=hvf \
         -cpu Haswell-v4 -smp 2 -m 4G \
         -device intel-hda -device hda-output \
         -device qemu-xhci \
         -device virtio-vga-gl \
         -device usb-kbd \
         -device virtio-net-pci,netdev=net \
         -device virtio-mouse-pci \
         -display cocoa,gl=es \
         -netdev user,id=net,ipv6=off \
         -drive "if=virtio,format=raw,file=hdd.raw,discard=on"
```


## Usage - Advanced

This section has additional configuration you may want to do to improve your workflow


### Clipboard sharing

There's now support for sharing clipboard in both directions: from vm->host and host->vm. To enable clibpoard sharing, add this to your command line:

```
         -chardev qemu-vdagent,id=spice,name=vdagent,clipboard=on \
         -device virtio-serial-pci \
         -device virtserialport,chardev=spice,name=com.redhat.spice.0
```

### Mouse integration

By default, you have mouse pointer capture and have to release mouse pointer from the VM using keyboard shortcut. In order to have seamless mouse configuration,
add the following to your command line instead of `-device virtio-mouse-pci`:

```
	-device usb-tablet \
```

### MacOS native networking for VMs (vmnet)

akihikodaki's patch set includes support for vmnet which offers more flexibility than `-netdev user`, and allows higher network throughput. (see https://github.com/akihikodaki/qemu/commit/72a35bb6e0a16bb7d346ba822a6d47293915fc95).

For instance, to enable bridge mode, replace:

```
    -device virtio-net-pci,netdev=net \
    -netdev user,id=net,ipv6=off \
```

with


```
    -netdev vmnet-bridged,id=n1,ifname=en0 \
    -device virtio-net-pci,netdev=n1 \
```

*(Note: Replace `en0` with your actual primary network interface if different. You can often find this in System Settings -> Network or by running `route -n get default | grep interface:`)*

vmnet also offers "host" and "shared" networking modes:

```
   # Host Mode (VM can talk to host and other host-mode VMs)
   -netdev vmnet-host,id=net0
   -device virtio-net-pci,netdev=net0

   # Shared Mode (VM uses NAT to talk to external network via host)
   -netdev vmnet-shared,id=net0
   -device virtio-net-pci,netdev=net0
```

***vmnet caveats:***

1) `vmnet-bridged` requires running qemu as root (`sudo`). `vmnet-host` and `vmnet-shared` generally do not.
2) Current vmnet API (Apple) doesn't support setting MAC address, so it will be randomized every time the VM is started.

To work around 2), for now it's possible to set the MAC address within the guest OS. For example, on Linux using systemd-networkd, create a `.link` file (e.g., `/etc/systemd/network/10-persistent-net.link`) for your virtual interface:

```ini
[Match]
# Match the virtual interface, e.g., by driver
Driver=virtio_net

[Link]
NamePolicy=kernel database onboard slot path
MACAddress=00:11:22:33:44:55 # Set your desired stable MAC address
```

Reboot the guest or restart `systemd-networkd.service` for the change to take effect. Adapt the `[Match]` section and MAC address as needed.


### Example: macOS AArch64 with HVF + Virgl GPU + Bridged Networking

This command launches an AArch64 Linux VM on an Apple Silicon Mac, utilizing the Hypervisor Framework (`accel=hvf`), Virgl 3D acceleration (`virtio-gpu-gl-pci`, `gl=es`), and bridged networking via `vmnet-bridged`.

**Requires running with `sudo`**. You'll need to provide your own kernel/initrd (if applicable) and disk image. Ensure you have the EDK2 firmware files (`edk2-aarch64-code.fd`, `edk2-arm-vars.fd`) copied to the working directory.

```sh
# Example assumes firmware files are in the current directory
# Replace kernel path and disk path with your actual files
# Replace en0 if your network interface is different

sudo qemu-system-aarch64 \\
    -machine virt,accel=hvf,highmem=off \\
    -cpu host \\
    -m 4G \\
    -smp 4 \\
    -drive "if=pflash,format=raw,file=./edk2-aarch64-code.fd,readonly=on" \\
    -drive "if=pflash,format=raw,file=./edk2-arm-vars.fd" \\
    -kernel path/to/your/linux_kernel \\
    -append "root=/dev/vda rw console=ttyAMA0" \\
    -drive "if=virtio,format=raw,file=./disk.raw,id=hd0" \\
    -device virtio-blk-pci,drive=hd0 \\
    -netdev vmnet-bridged,id=net0,ifname=en0 \\
    -device virtio-net-pci,netdev=net0 \\
    -device virtio-gpu-gl-pci \\
    -display cocoa,gl=es \\
    -device qemu-xhci \\
    -device usb-tablet \\
    -device usb-kbd
    # Add other devices like sound as needed: -device intel-hda -device hda-output
```
