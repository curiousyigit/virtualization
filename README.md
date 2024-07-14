# Virtualization
This repository hosts my virtualization setups and configurations. You are welcome to fork it and modify it to your needs

# QEMU/KVM

## Systems
### System 1: Acer Nitro 5 AN517-54-56H8 (Laptop)
- <b>CPU:</b> Intel Core i5-11400H @ 2.70GHz
- <b>iGPU:</b> Intel UHD Graphics (/dev/dri/renderD128)
- <b>dGPU:</b> NVIDIA GeForce GTX 1650 4GB with NVIDIA proprietary driver (/dev/dri/renderD129)
- <b>RAM:</b> 40GB DDR4 (8GB + 32GB)
- <b>SSD:</b> 2TB NVMe M.2 SSD (7,300MB/s read) WD Black SN850X
- <b>Host OS:</b> Fedora KDE 41
    - glmark2 score with iGPU: 7250
    - glmark2 score with dGPU: 4575 with error (lower than iGPU for some reason)

## VirGL Support
### VirGL Intel iGPU
VirGL with an Intel iGPU seems to work out of the box with no flickering and smooth graphics

### VirGL NVIDIA dGPU
To use VirGL with NVIDIA GPU in some distros (e.g. Fedora), you need to give libvirtd access by editing /etc/libvirtd/qemu.conf and uncommenting
```
cgroup_device_acl = [
    "/dev/null", "/dev/full", "/dev/zero",
    "/dev/random", "/dev/urandom",
    "/dev/ptmx", "/dev/kvm",
    "/dev/userfaultfd",
    "/dev/nvidiactl", "/dev/nvidia0", "/dev/nvidia-modeset", "/dev/dri/renderD129", "/dev/dri/by-path/pci-0000:01:00.0-render"
]
```
- Note: we are only interested in the last line in the example above, if for some reason in your system the lines above `"/dev/nvidiactl", ...` are different, let them be and only copy this line from here
- Modify `/dev/nvidia0`, `/dev/dri/renderD129` & `/dev/dri/by-path/pci-0000:01:00.0-render` according to your system and where the nvidia gpu is located
- Save the file and restart libvirtd by running `systemctl restart libvirtd`

## Multiscreen Support
Multiscreen only works when not using OpenGL within Spice
```
<graphics type="spice">
  <listen type="none"/>
  <image compression="off"/>
  <gl enable="no"/>
</graphics>
```
You can specify the number of screens (heads) in the Virtio config
```
<video>
  <model type="virtio" heads="2" primary="yes">
    <acceleration accel3d="yes"/>
  </model>
  <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x0"/>
</video>
```
Spice by-itself does not support multiple screens. To have multiple screens, you can use another tools (e.g. `virt-viewer`)
- Run `sudo virt-viewer -a`
- Select your VM. You will have 2 or more windows according to the heads you specified
- <b>Note:</b> Spice within `virt-manager` will automatically close. If you close `virt-viewer`, you can simply click on the "info" icon in `virt-manager` and then click again on the "monitor" icon to get Spice running again

## 3D Acceleration (OpenGL)
### OpenGL within Spice (Intel iGPU)
OpenGL within spice (Intel iGPU) works great with no vertical flickering, however we cannot add more than one screen using `virt-viewer` which doesn't even work
```
<graphics type="spice">
    <listen type="none"/>
    <image compression="off"/>
    <gl enable="no"/>
    <gl enable="yes" rendernode="/dev/dri/renderD128"/>
</graphics>
...
<video>
  <model type="virtio" heads="2" primary="yes">
    <acceleration accel3d="yes"/>
  </model>
  <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x0"/>
</video>
```
- System 1 (Fedora KDE 41 guest)
  - glmark2 score: 226
  - No desktop vertical flickering
  - No Youtube video vertical flickering

### OpenGL within Spice (NVIDIA dGPU)
OpenGL within spice (NVIDIA dGPU) does not work and crashes `virt-manager`. The VM starts but then quickly shuts off
```
<graphics type="spice">
    <listen type="none"/>
    <image compression="off"/>
    <gl enable="no"/>
    <gl enable="yes" rendernode="/dev/dri/renderD129"/>
</graphics>
...
<video>
  <model type="virtio" heads="2" primary="yes">
    <acceleration accel3d="yes"/>
  </model>
  <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x0"/>
</video>
```
- https://github.com/virt-manager/virt-manager/issues/661
### OpenGL as egl-headless (Intel iGPU)
OpenGL oustide of Spice (Intel iGPU) works okay with some vertical flickering but we can add multiple screens using `virt-viewer`. Youtube videos seem to work okay however and not suffer from flickering
```
<graphics type="spice">
    <listen type="none"/>
    <image compression="off"/>
    <gl enable="no"/>
</graphics>
<graphics type="egl-headless">
    <gl rendernode="/dev/dri/renderD128"/>
</graphics>
...
<video>
  <model type="virtio" heads="2" primary="yes">
    <acceleration accel3d="yes"/>
  </model>
  <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x0"/>
</video>
```
- System 1 (Fedora KDE 41 guest)
  - glmark2 score: 206
  - Desktop vertical flickering
  - No Youtube video vertical flickering

### OpenGL as egl-headless (NVIDIA dGPU)
OpenGL oustide of Spice (NVIDIA dGPU) works okay with some vertical flickering but we can add multiple screens using `virt-viewer`. Youtube videos seem to work okay however and not suffer from flickering
```
<graphics type="spice">
    <listen type="none"/>
    <image compression="off"/>
    <gl enable="no"/>
</graphics>
<graphics type="egl-headless">
    <gl rendernode="/dev/dri/renderD129"/>
</graphics>
...
<video>
  <model type="virtio" heads="2" primary="yes">
    <acceleration accel3d="yes"/>
  </model>
  <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x0"/>
</video>
```
- System 1 (Fedora KDE 41 guest)
  - glmark2 score: 3759
  - Desktop vertical flickering
  - No Youtube video vertical flickering

## Shared Folder
To share a folder with the host, you need shared memory
1. Go to "Memory" section of `virt-manager` and enable "shared memory"
    ```
    <memoryBacking>
      <source type="memfd"/>
      <access mode="shared"/>
    </memoryBacking>
    ```
2. Add a new Filesystem hardware (virtiofs) and define your source path. For your target path, enter a name (e.g. shared)
    ```
    <filesystem type="mount" accessmode="passthrough">
      <driver type="virtiofs"/>
      <source dir="/mnt/shared/virtual-machines/rtw/shared"/>
      <target dir="shared"/>
      <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
    </filesystem>
    ```
3. Run your guest and edit `/etc/fstab` and add the line `shared /mnt/shared virtiofs defaults 0 0`
4. Reboot your guest
5. You can now access the shared folder in the guest in `/mnt/shared`