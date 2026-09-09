## Contents
- [Overview](#-Overview)
- [Goal](#-goal)
- [Prerequisites](#-prerequisites)
- [1. BIOS/UEFI Settings](#-step-1-biosuefi-settings)
- [2. Physical Monitor Connection](#-step-2-physical-monitor-connection)
- [3. Kernel Parameters](#️-step-3-kernel-parameters)
- [4. Configure Looking Glass Client on Arch](#-step-4-configure-looking-glass-client-on-arch)
- [5. Configure Looking Glass IVSHMEM in VM XML ](#-step-5-configure-looking-glass-ivshmem-in-vm-xml)
- [6. Windows Guest Configuration](#step-6-windows-guest-configuration)
- [7. Before Running the Scripts](#️-step-7-before-running-the-scripts)
- [8. The Scripts (Wrapper + hook scripts)](#-step-8-the-scripts-wrapper--hook-scripts)
- [9. Additional VM Tuning](#️-step-9-additional-vm-tuning)
- [10. Workflow Configuration and Running Looking Glass Client](#-step-10-workflow-configuration-and-running-looking-glass-client)
- [Final Summary / Check list](#-final-summary--check-list)
- [Troubleshooting Tips](#-troubleshooting-tips)
- [Test the Workflow](#-test-the-workflow)

</br>

## Overview
In this guide we will learn how to do GPU passthrough with an ***iGPU + dGPU*** setup without being logged out of your current DE/WM session.

The scripts in this guide are meant for **Hybrid Graphics** systems. The scripts and guides were inspired by the Single GPU Passthrough scripts over in the *[RisingPrisimTV](https://gitlab.com/risingprismtv/single-gpu-passthrough)* guide, which is meant for a single--literally only 1 GPU--setup. 

Since we have integrated graphics, *our setup is much more flexible and doesn't require the sledgehammer approach* in the RisingPrisimTV scripts. In fact, many of the bugs and mystery problems that inspired this guide may have been from me trying to forcefully use those scripts for my hybrid GPU setup.

> NOTES: 
> 1. This guide assumes you are somewhat familiar with the VFIO/GPU-Passthrough process, that you can install the libivrt, qemu/kvm stack for your distro, and are comfortable manually editing the scripts for your setup needs.
> 2. (References) If you don't know anything, you might want to try starting here:
>       - https://passthroughpo.st/simple-per-vm-libvirt-hooks-with-the-vfio-tools-hook-helper/
>       - https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF
>       - https://gitlab.com/risingprismtv/single-gpu-passthrough/-/wikis/home
>

</br>

---

## 🎯 Goal
A frictionless workflow where you can:
1.  Game on Arch Linux using `prime-run` (Nvidia renders, Intel displays).
2.  Start a Windows VM that takes exclusive control of the Nvidia dGPU for hardware-accelerated Looking Glass.
3.  Shut down the VM and instantly return to Linux host (**no reboot, no logout, no frozen screens)**.

---

## 📋 Prerequisites
* More than 1 monitor/display
* (Recommended) Arch Linux with KDE Plasma 6.6+ Wayland.
* `qemu`, `libvirt`, `virt-manager` installed and working.
* Your Windows 11 VM already created and functional *(Just with SPICE graphics for now).*

---

## 🔧 Step 1: BIOS/UEFI Settings
 
 Reboot and enter your BIOS/UEFI and make these changes. Some are a hard-requirement to allow for passthrough, and others are to maximize both VM stability and native Linux gaming performance.
 
1. **(Required) Primary Graphics Adapter:** `[IGD]`
    - (NOTE) For this to take effect you might need to shutdown completely once, after enabling it
2. **(Required) Intel VT-D:** `[Enabled]`
3. **(Required) CSM (Compatibility Support Module):** `[Disabled]` _(Crucial. VFIO require pure UEFI for passthrough. Also required by ReBAR)._
4. **IGD Multi-Monitor:** `[Enabled]` *(This option seems to disappear when set to `IGD`)*
5. **IGD Shared Memory:** `[64M]` or `[128M]`
6. _(Optional)_ **Above 4G Decoding:** `[Enabled]`
7. _(Optional)_ **Re-Size BAR Support:** `[Enabled]`
8.  Save and exit.

> **NOTES:**
> 1. **(IMPORTANT)** *Above 4G Decoding* and *Re-Size BAR Support* can cause hangs when powering off on some setups. If you experience this try switching to the open source NVIDIA modules, adjust your kernel boot paremeters, or **Just leave disabled** (if the extra performance isn't worth the hassle).
> 2. (MSI Click BIOS 5) When I enabled `Above 4G Decoding` and `Re-Size BAR Support` and hit *Save & Exit* the confirmation screen showcased that it also enabled *"fast boot"*, It set `Integrated Graphics Share Memory` back to *[32M]*, and it enabled something called `Above 4GB MMIO BIOS Assignment`. I handled this by: Disabling *"fast boot"* again, setting the iGPU share memory back to *64M*, and then saving and exiting.
> 
> 3. **Above 4G Decoding** tells the _Linux Kernel_ that it is allowed to map the GPU's memory above the 4GB boundary.
> 4. **Above 4GB MMIO BIOS Assignment** tells the _Motherboard BIOS_ to actually allocate that physical address space during the boot process.
>

</br>

---

## 🔌 Step 2: Physical Monitor Connection
*   **iGPU Monitor:** Plug your monitors to the iGPU only. In other words, into the **motherboard's HDMI/DisplayPort**.
* **dGPU Monitor:** Unplug all monitors. 
> NOTE: If you **HAVEN'T** configured your system for Early KMS, then you can have a monitor plugged in--in fact we will need it later in the guide.

</br>

---

## ⚙️ Step 3: Kernel Parameters
1. Edit whatever controls your kernel parameters. For me, it was *UKI*, for you it might be GRUB. (e.g. UKI: `/etc/kernel/cmdline`)

2. **(/!\ CRUCIAL /!\\)** Delete these two parameters:
   Remove `nvidia_drm.modeset=1` and `nvidia_drm.fbdev=1`.
   Your line should now look something like this:
   ```text
   root=PARTUUID=<UUID> zswap.enabled=0 rw rootfstype=ext4 intel_iommu=on iommu=pt
   ```
3. **Update your kernel parameters:**
   ```bash
   sudo grub-mkconfig -o /boot/grub/grub.cfg
   sudo mkinitcpio -P
   ```
4. **Reboot your PC.**

> NOTES:
> - we need to remove `nvidia_drm.modeset=1` from the kernel parameters to prevent KWin from holding the dGPU. Without this step, the VM will hang.
> - Do **NOT** add `vfio-pci.ids=...` or any VFIO binding parameters here. We want the `nvidia` driver to load normally at boot so you can game on the host. The hook scripts will handle dynamic switching.
>

5. If you use secure boot, check that EFI images are still signed.
```bash
sudo bootctl status
sudo sbctl status  #check that your EFI images are signed
sudo sbctl verify
```

</br>

---

## ⚙️ Step 3.5: (OPTIONAL) Hiding Your dGPU-Connected Monitor and Avoiding the Windows "Headless GPU" Problem

> **NOTE:** While this is technically "optional," If you continue WITHOUT a monitor, you'll get a headless GPU scenario where Windows sees the GPU but doesn't use it. It's only optional in the sense that, you can get your dGPU to a state where nothing is holding it by unplugging everything.

**The main problem:** 
1. If you boot the windows vm with nothing plugged into the NVIDIA card, _windows will enter a headless gpu mode that doesn't let you configure any display settings_ (even with the NVIDIA drivers installed).
2. However, _if you plug a monitor into the dGPU while the VM isn't running, `kwin` will immediately pick it up as a viable display output_ (will appear in display settings).  

### --- Stop Linux from grabbing the newly plugged-in dGPU monitor

We can pass `nvidia-drm.modeset=0` as a kernel boot parameter.
Setting `modeset=0` disables Kernel Mode Setting (KMS) for the NVIDIA driver. It tells the Linux kernel: _"Do not configure display outputs, resolutions, or monitors for this hardware at the kernel level."_

1. Remove any Early KMS configuration changes from:
    1. `/etc/mkinitcpio.conf`
    2. `/etc/default/grub`  
    > NOTE: In my setup grub is just a boot menu and doesn't set kernel parameters, and having them in `mkinitcpio.conf` seems to not matter... BUT - to avoid  future problems might as well remove them.

2. **(CRUCIAL)** Set `nvidia_drm modeset=0` and `fbdev=0` in `/etc/modprobe.d/nvidia.conf` 
-- this effectively disables these modules.

3. Apply the changes: 
```bash
#1. sudo mkinitcpio -P 
#2. sudo grub-mkconfig -o /boot/grub/grub.cfg
#3. reboot
```

</br>

---

## 🪟 Step 4: Configure Looking Glass Client on Arch

[See My Looking Glass Guide: Installing](https://github.com/omin4/looking-glass-guide-arch-linux#1-installing)

</br>

---

## 🪟 Step 5: Configure Looking Glass IVSHMEM in VM XML

[See My Looking Glass Guide: Configure Libvirt / QEMU XML](https://github.com/omin4/looking-glass-guide-arch-linux#3-configure-libvirt--qemu-xml)

</br>

---

## Step 6: Windows Guest Configuration (for Looking Glass)

[See My Looking Glass Guide: Setting Up the Windows Guest](https://github.com/omin4/looking-glass-guide-arch-linux#4-setting-up-the-windows-guest)

</br>

---

## ⏸️ Step 7. Before Running the Scripts

> NOTES: 
> 1. Enable `sysrq` keys for REISUB just in case. Detaching a graphics card can be finicky; things can happen.
> 2. _(Recommend)_ Run the scripts from the terminal the first time (without the VM). This avoids `libvirtd` locking up if a problem occurs. (Below commands are useful for logging exactly what happens)
>     ```bash
>     sudo stdbuf -oL -eL bash -x /etc/libvirt/hooks/qemu.d/win11/prepare/begin/vfio-startup > starthook.log
>     
>     sudo stdbuf -oL -eL bash -x /etc/libvirt/hooks/qemu.d/win11/release/end/vfio-teardown > endhook.log
>     ```
>

**PREFACE:** The **reality** is, even though we set `nvidia_drm modeset=0`, apps and other background processes will still probe and target the NVIDIA device files (`/dev/nvidia0` and `/dev/nvidia-uvm`, etc) for things such as video rendering or AI computation. An infamous example is any Electron app e.g. Obsidian, which uses hardware acceleration by default (you can disable in the appearance settings).

> **⚠️ If you attempt to detach the GPU while an app is background-probing it, the unbind command will hang indefinitely, forcing a reboot.**

1. **(IMPORTANT) Double check nothing is running on the dGPU with the following commands:**
    1. `nvidia-smi`
    2. `sudo fuser -v /dev/dri/card*`
    3.  `sudo fuser -v /dev/nvidia*`
    <details>
    <summary><b> (Click to expand)...The output should look like this... </b></summary>

    ```bash
    $ nvidia-smi
    =========================================================================================|
    |  No running processes found                                                             |
    +-----------------------------------------------------------------------------------------+
    
    $ sudo fuser -v /dev/dri/card*  # card2 =/= NVIDIA
                         USER        PID ACCESS COMMAND
    /dev/dri/card2:      root          1 F.... systemd
                         root        664 F.... systemd-logind
                         user1      1131 F.... kwin_wayland
                         user1      1219 F.... Xwayland
     
    $ sudo fuser -v /dev/nvidia*   # BAD Output!!!
                         USER        PID ACCESS COMMAND  
    /dev/nvidia0:        archy     51681 F...m kclockd  
    /dev/nvidiactl:      archy     51681 F...m kclockd  
                         archy     71381 F.... obsidian  
    /dev/nvidia-uvm:     archy     51681 F.... kclockd  
      
    $ sudo fuser -k /dev/nvidia*  # Kill the BAD Output
    /dev/nvidia0:        51681m  
    /dev/nvidiactl:      51681m  
    /dev/nvidia-uvm:     51681  
      
    $ sudo fuser -v /dev/nvidia*  # GOOD Output
    
    ```

    </details>

</br>

2. **After running checks:** if something is using dGPU, **KILL IT!** (Apps, kwin, etc.) One way to achieve this is by using `fuser` to Kill processes accessing the device file.
    ```bash
    sudo fuser -k /dev/nvidia* 
    sudo fuser -k /dev/dri/card<#>
    # To find your "card-" number run this command:
    ls -la /sys/class/drm/card0/device/driver
    ```
    - The first command kills everything accessing any *nvidia\* device file* in the liniux device directory (in Linux, almost "everything is a file").
    - The second command kills anything using your dGPU-connected display. 

> NOTE: **`card0`, `card1`, etc.**: These represent the primary legacy display pipelines (often called "legacy/KMS nodes") used for display output, screen configuration, and buffer management.

3. **If you DON'T have a monitor plugged in to the dGPU yet, plug one in** (your DE should't detect/capture the new display). Run checks again. 

4. Add the qemu file and scripts to `/etc/libvirt/hooks/` and make sure they're executable (`chmod +x`).
> **NOTES:** 
> 1. You can follow the  *RisingPrisim* method for settings up the hook scripts for the most part, but some parts are outated, such as:
>     1. They enable the legacy `libvirtd` daemon instead of the newer, modular `virtqemud` daemon. This can cause problems down the line.
>     2. I use my own edited `qemu` file and `nosleep` file.)
>

5. Add ALL of your NVIDIA PCI devices to your VM. This can be done easily through `virt-manager`.

### qemu file & nosleep service file

___
> [libvirt-nosleep.service ->](libvirt-nosleep.service)

> __NOTES:__ 
> 1. Save as a *.service* file to: 
>     ```
>     /etc/systemd/system/libvirt-nosleep@.service
>     ```
> 2. ***CHECK*** if nosleep service running after vm start: 
>     1. `systemd-inhibit --list` or,
>     2. check systemctl status logs: 
>     ```
>     systemctl status libvirt-nosleep@<vm-name>.service
>     ```
>

</br>

> [qemu file ->](qemu)

> __NOTE:__ Make executable: `sudo chmod +x /etc/libvirt/hooks/qemu`

</br>

---

## 📜 Step 8: The Scripts (Wrapper + hook scripts)

> **NOTES:**
>
> 1. Current environment configuration at the time of writing: 
>     1. KDE Plasma 6.6.5 wayland on Arch linux, 
>     2. I've set Set `nvidia_drm modeset=0` and `fbdev=0` in `/etc/modprobe.d/nvidia.conf` so that kwin doesn't hold my NVIDIA dGPU when a monitor is directly connected. 
>         - *(This has largely worked -- kwin doesn't detect the monitor connected directly to the dGPU.)*
> 2. **(REQUIRED)** For the virsh command to work `user = "your username"` and `group = "your username"` added to  `/etc/libvirt/qemu.conf`
> 3. Having both safety checks in the wrapper script and script itself may seem redundant or overkill, but they're all to **avoid a virtqemud/libvirtd hang** -- since it's almost always requires a reboot. 
>     - *(The nuclear option is a last-ditch effort to this aim, as we want to avoid a hang at all costs!)*
>     - The wrapper adds a layer of separation and lets you close the offending apps manually in cases where a sudden exit would lose data
> 4. If you don't want the "nuclear option" just remove the final check block -- or replace it with the block below that simply exits (`virtqemud` hang likely)
>    ```bash
>    # Re-verify that the locks are actually gone
>    if fuser -s /dev/nvidia* 2>/dev/null || fuser -s "$NVIDIA_DRI_CARD" 2>/dev/null; then
>        log "FATAL: Auto-kill failed. Processes are still holding the GPU. Aborting to prevent panic."
>        systemctl start nvidia-persistenced 2>/dev/null
>        exit 1
>    fi
>    ```
> 5. **(WARNING)** Don't place multiple executable files in the hook folders (e.g. `/release/end/teardown.bak`), as the qemu file will attempt to execute all files in this dir. 
>
> 

</br>

### Wrapper Script

> **NOTES:**
> - **DON'T** run the wrapper script with `sudo`
> - To avoid a permission error, **make the log file writable by everyone** (safe):
>     - `sudo chmod 666 /var/log/libvirt/vfio-hook.log`
> - **You may get an error** `Failed to get domain 'win11'` -- this is not a permission error, but a connection routing error. 
>     - **WHY:** Libvirt has two distinct "spaces" for virtual machines: **`qemu:///system`** and **`qemu:///session`** *(When you type `virsh` as a normal user, it defaults to 'qemu:///session')*. Because your normal user is looking in the `session` space, it simply cannot find your VM's.
>     - **FIX:** Just tell `virsh` to look in the `system` space. You can do this with commands above, or set an environment variable in `.bashrc`:
>     ```
>     export LIBVIRT_DEFAULT_URI="qemu:///system"
>     ```
>

1. Add the **STOP/Teardown Alias** to your `.bashrc`:

```
alias vmstop='virsh -c qemu:///system shutdown win11-4 && echo "⏳ Sending shutdown signal to Windows..."'
```

2. Add the **Start Script Wrapper** to your bin path and name it `vmstart`
    > ___
    > [vmstart ->](vmstart)
    > ___

</br>

### ▶️ Start Script

> **NOTES:**
> - **Possible User Error 1:** Using the old, legacy `libvirtd` daemon.
>         ```
>         error: failed to connect to the hypervisor
>         error: Failed to connect socket to '/var/run/libvirt/virtqemud-sock': No such file or directory
>         ```
>     - **Reason:** Because the RisingPrisimTV Guide is a bit out of date.
> - **Possible User Error 2:** _Use .socket instead of .service_ - "socket activation" for libvirt daemons and related services is preferred so they only turn on when needed, saving RAM. Enabling the `.service` forces the network daemon to stay running in the background 24/7.
> - **Possible User Error 3:** _You no longer need to unload the PCI devices_ - Our new start script only needs to manage the Nvidia kernel modules so the kernel doesn't panic.
>     - Because the RisingPrisimTV Guide is old, we were using an outdated method to detach our PCI devices, back when  `managed='yes'` was not as robust and required scripts to detach them.
>     - Libvirt handles the vfio-pci binding automatically via modern `<hostdev ... managed='yes'>` in the VM XML. Thus the `virsh nodedev-detach` command is redundant and only clutters logs when it fails.
> **Possible User Error 4:** Didn't make the `qemu` file executable.
> 
> - **Added a "Filter"** to the script to protect our display-manager and other system services from being killed by the "auto-kill" feature, and instead killed in a more controlled manner by the "nuclear" option codeblock. 
>

</br>

Replace the contents of the *vfio-startup* with this streamlined version: 
`/etc/libvirt/hooks/qemu.d/win11/prepare/begin/vfio-startup` 

___
> [vfio-startup script ->](vfio-startup)

> NOTES: 
> - **Check script logs:** `tail -f /var/log/libvirt/vfio-hook.log`
> - This script has a **Safety Check** that prevents kernel panics. It checks to see if any NVIDIA modules are still loaded *(like in the case of kwin holding `nvidia_drm` when I had `nvidia_drm.modeset=1` in kernel parameters)*. If a background app is still using the GPU, the script attempt to kill that process, if it's not able to, it logs you out of your session by killing the display-manager (preventing the hang). Last, it attempts to abort with `exit 1` (this will likely cause a virtqemud/libvirtd hang because of a libvirt bug?)
>
___

### ⏹️ End Script
> - started by an alias in `.bashrc`
> - Has logic that handles if the "nuclear option" was triggered
> - Has an if statement that checks if the NVIDIA modules are already loaded.

Replace the contents of `/etc/libvirt/hooks/qemu.d/win11/release/end/vfio-teardown` with this. 
It reattaches the GPU and reloads the Nvidia driver -- nothing more.

___
>[vfio-teardown ->](vfio-teardown)
___

</br>

---

## ⚙️ Step 9: Additional VM Tuning

1. Find the `<memballoon>` tag and set its type to `none`

2. CPU pinning. (Remember: view __Physical Cores__ and __Threads__ as pairs, e.g. `0,6`, `1,7`, `2,8`, etc...

3. Add virtio mouse & keyboard:
    1. - Create an `<input type='mouse' bus='virtio'/>` device, if you don’t already have one.
    2. Create an `<input type='keyboard' bus='virtio'/>` device to improve keyboard usage.

4. Be sure to set your CPU model type to `host-passthrough` so that your guest operating system is aware of the acceleration features of your CPU and can make full use of them.

</br>

---

## 🔀🪟 Step 10: Workflow Configuration and Running Looking Glass Client

> - We'll be using the `virsh` terminal command to start and stop the VM.

1. Let's change the capture key from the default `scrLK` to something else and add a few QoL command line options.
    1. Create: `vim ~/.config/looking-glass/client.ini`
    2. Add:
    ```ini
    [win]
    fullScreen=yes
    autoResize=yes
    noScreensaver=yes      ; Crucial: Prevents screen from turning off during long sessions
    quickSplash=yes        ; Removes the 2-second fade-in animation when alt-tabbing
    alerts=no              ; Hides annoying pop-up notifications
    
    [input]
    escapeKey=KEY_GRAVE    ; The `~` key. Hold it to see the menu, tap it to toggle capture.
    captureOnFocus=yes     ; Clicking window instantly captures the mouse!
    grabKeyboard=yes       ; Sends all media keys/special keys directly to Windows
    hideCursor=yes         ; Hides the Linux mouse cursor when captured
    rawMouse=yes           ; 1:1 pixel-perfect tracking
    
    [spice]
    enable=yes
    clipboard=yes          ; Copy/paste between host and guest	
    audio=no               ; CRITICAL: Disables virtual audio to prevent ASIO driver conflicts
    
    [audio]
    micDefault=deny        ; Prevents Windows from accidentally grabbing your Linux mic
    
    ```

2. Create a workflow cheatsheet:
```bash
🔀 WORKFLOW CHEAT SHEET
-----------------------------------
Launch vm : vmstart
Stop vm   : vmstop
Help Menu     : vmhelp

🪟 LOOKING GLASS KEYBINDS
(Press the [ ~ / ` ] key to toggle mouse capture. HOLD it to access shortcuts)
-----------------------------------
Toggle Capture: [ ~ ] (Grave/Backtick key)
Fullscreen    : [ ~ ] + F
Quit LG       : [ ~ ] + Q
Volume Up/Down: [ ~ ] + F12 / F11
Mute Guest    : [ ~ ] + F10

```
   
</br>

---

## 🎉 Final Summary / Check list
- [x] BIOS: IGD Primary
- [x] Arch Host: KDE Plasma running on Intel iGPU, Nvidia driver idle
- [x] Kernel: Removed `nvidia_drm.modeset=1` to prevent KWin from holding the dGPU
- [x] Scripts: Dynamic start/teardown scripts working perfectly
- [x] VM: PCIe USB controller passed for Saffire (zero audio latency)
- [x] VM: Nvidia GPU passed for hardware-accelerated Looking Glass
- [x] Windows "headless GPU" issue fixed by plugging a monitor into dGPU or software solution.

</br>

---

## 🚨 Troubleshooting Tips

* __Double check nothing is running on the dGPU__
    1. `nvidia-smi`
    2. `sudo fuser -v /dev/dri/card*`
    3.  `sudo fuser -v /dev/nvidia*`

</br>

* Unplugged anything from the dGPU

</br>

*   **VM hangs on start:**  Check `/var/log/libvirt/vfio-hook.log`.

</br>

*   **Looking Glass shows a black screen:** Ensure the IVSHMEM block is in your VM XML. Ensure the Looking Glass host is running in Windows. Ensure your user is in the `kvm` group.
  
</br>

*   **Audio dropouts in VM:** Ensure you pinned CPU cores for the VM in `virt-manager` (CPU topology -> pinning). Ensure the PCIe USB controller is passed through, not just the USB device.

</br>

---

## Test the Workflow

1. Open your Arch terminal and run `looking-glass-client`.

2.  **Test Audio** 

3.  **Test VM Shutdown:** Shut down the Windows VM. Wait ~5 seconds. Run `nvidia-smi` on Arch. The Nvidia card should be back, and you should be able to launch a game with `prime-run` immediately.

4.  **Test Gaming:** Launch a game with `prime-run %command%` in Steam. Confirm it renders on Nvidia and displays on your Intel-connected monitor.

</br>
