# Batocera-launcher
Enter/Exit Batocera from the host operating system (for Raspberry Pi 5)

The Batocera OS is highly optimized for games, as a result, it has left out all other capabilities a normal operating system has. This makes life a bit difficult for users who wants both playing games and doing computing stuffs, because you need to install a separate operating system, need to buy and flash a separate microSD/SSD/USD-HDD, and reboot in order to switch between operating systems.

In this tutorial, we describe how to directly run Batocera’s EmulationStation from inside Raspbian OS on Raspberry Pi 5 without rebooting or flashing MicroSD card. The same principle can be applied to PC or other systems (such as Ubuntu) as well. My previous solution no longer works on new versions of Batocera and Raspbian OS, thanks to OpenAI ChatGPT which helped me figured out this new elegant solution which does not require modifying Batocera root filesystem contents and can handle Batocera future updates for at least several generations.

# Preparation (need to do once)
In order to run Batocera emulationstation inside Raspbian OS 64-bit (on RPi5), you need to make some adjustments to the default OS.
1. Switch kernel to `kernel8.img`. Raspbian OS has two kernels: `kernel_2712.img` and `kernel8.img`. The former uses 64K page alignment while the latter uses 4K page alignment. By default, the OS uses `kernel_2712.img` with some slight speed advantage (~7%). Since Batocera Linux (up to version 41) uses the default 4K page alignment, without switching to 4K-page-alignment kernel, chroot into Batocera will immediately throw segmentation fault. To do so, modify (or add if absent) the following into `/lib/firmware/config.txt`:
```
[all]             
kernel=kernel8.img
```

2. Switch sound system from `pipewire` to `alsa`. Batocera uses `alsa` sound system, which is not compatible with Raspbian OS default `pipewire` sound system. To do so, set `dtparam=audio=on` in `/lib/firmware/config.txt`, and run the following and reboot:
```
systemctl --user mask pipewire.socket pipewire.service \
                      pipewire-pulse.socket pipewire-pulse.service \
                      wireplumber.service
systemctl --user stop pipewire.socket pipewire-pulse.socket wireplumber.service
```
After reboot, make sure sound is still working on the host system, if not, `apt install alsa-base`.

3. Configure bluetooth daemon to be stoppable. On many systems by default, `systemctl stop bluetooth` cannot stop bluetooth daemon because the service is configured to keeps respawning to workaround some bugs which can cause bluetooth daemon to crash and bluetooth will stop working. Edit `/lib/systemd/system/bluetooth.service` and set `Restart=on-failure`.

# Main steps
The steps are explained in details below:
1. Download and extract Batocera Image for Raspberry Pi 5, you can also download pre-made Batocera game-pack images from ArcadePunk. This is typically `XXX.img.xz` or `XXX.img.gz` or etc. Extract the image so that you have `XXX.img`.
2. Download the script `bato-launch.sh`.
3. SSH into the host system.
4. To start Batocera, run `./bato-launch.sh <your-batocera-image.img>`, to stop Batocera, run `./bato-launch.sh stop`.

# Details explained (`bato-launch.sh`)
All steps are included in the script `bato-launch.sh`. Here, I will provide detailed explanations.
1. Firstly, use `losetup` and `mount` to mount the Batocera image and its first partition.
2. Mount Batocera root file-system as overlay so that we can write to it. Modifications will be stored in `upperdir`.
3. Bind mount device folders (`dev dev/pts`), system runtime folders (`proc sys run var/run`), and driver folders (`lib/firmware`) because we are running Raspbian kernel instead of Batocera Linux kernel.
4. Copy over `/etc/resolv.conf` so that emulationstation can access Internet directly.
5. If it is the first time, copy over initial system configuration files which is necessary.
6. On Raspberry Pi, there are 2 GPU device slots, one for KMS, one for FKMS. Upon different reboots, these 2 slots can be swapped. So we need to determine which GPU slot is active.
7. For the host system, stop display manager and bluetooth daemon, so that Batocera's display manager and bluetooth daemon can run.
8. Start all essential Batocera services in Batocera's SysVinit sequence, these includes seat daemon, audio daemon, emulationstation, bluetooth service, etc.

For quiting Batocera, simply undo each step in the reverse order.

# Work-in-progress
- keyboard does not work yet in emulationstation