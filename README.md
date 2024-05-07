# Ubuntu on a Lenovo Thinkbook laptop

My notes on how to set up a lenovo thinkbook gen6 laptop with ubuntu 24.04 as a hypervisor for QEMU/KVM virtualization with GPU passthrough

main challenges:

- lenovo by default disabled S3 sleeping state in favor of `modern standby` for the windows. moreover, lenovo hid/removed BIOS settings for switching it back. I couldn't make the newest kernel 6.8.0 work with this `modern standby`. so the other solution for normal everyday computer use was to use hibernation, sleeping state S4

- because of the bug in 24.04 it is not possible to use encrypted swap file for hibernation. swap file must be changed for an encrypted swap partition https://bugs.launchpad.net/ubuntu/+source/systemd/+bug/2057687

- all partitions except for the boot, must be encrypted with LUKS. including swap, that is used for hibernation.

- secure boot must be disabled in order to have hibernation and gpu passthrough working. this is a security risk, as it potentially gives an opportunity to decrypt the disks that are taken out and connected to another machine. (to do: look into recompiling the kernel to disable lockdown to keep the secure boot)

**current problems:**

- _must shut down guests with GPU passthrough before host hibernation._ if Ubuntu guest VM with GPU passthrough was running when host went to hibernation, the GPU in the guest is not working correctly after host wakes up from hibernation, the guest would require a reboot.

## install ubuntu 24.04 desktop host, initial setup and swap settings

- change bios settings to enable S4 (there is no option for S3)

  - use the code to enter the service version of lenovo BIOS setup
    - https://www.reddit.com/r/Lenovo/comments/zq3tc5/how_to_disable_modern_sleep_and_enable_s3_sleep/
    - under advanced, ACPI settings, disable acpi autoconfig, _enable hibernation_

- make bootable pendrive with https://github.com/ventoy/Ventoy
- download https://mirrors.tuna.tsinghua.edu.cn/ubuntu-releases/noble/ubuntu-24.04-desktop-amd64.iso and put into the root of the pendrive (or from https://mirrors.aliyun.com/ubuntu-cdimage/)
- install choosing `encrypted, LVM`
  - set one of the mirrors
    - https://mirrors.tuna.tsinghua.edu.cn/ubuntu/
    - https://mirrors.aliyun.com/ubuntu/
- reboot into live cd again to resize the logical volume for the root partition on the encrypted volume group, then add another logical volume for the new swap partition
  - press ctrl alt t
  - `lsblk` look for the name of the last partition
  - `cryptsetup open /dev/nvme0n1p3 crypt` it will ask for the passphrase
  - `lsblk`
  - `lvresize --verbose --resizefs -L -96G /dev/mapper/ubuntu--vg-ubuntu--lv` # free 150% of RAM size for the new swap partition, because there might be some funny fringe case involving video memory that would require more space on disk than the size of the actual RAM
  - `sudo e2fsck -f /dev/mapper/ubuntu--vg-ubuntu--lv`
  - `sudo lvdisplay`
  - `sudo lvm lvcreate ubuntu-vg -n swaplv -L 96G`
  - `reboot`
- boot into the installed system, change swap file for swap partition
  - `swapon --show`
  - `ls /dev/mapper` find the exact name of the mapper
  - `sudo mkswap /dev/mapper/ubuntu--vg-swaplv`
  - `sudo nano /etc/fstab` -` /dev/mapper/ubuntu--vg-swaplv   none    swap    sw      0       0`
  - `sudo swapon -a`
  - `swapon --show` or `free -m`
  - `sudo rm /swap.img`
  - add options for swap to the grub
    - `sudo nano /etc/default/grub`
      - ```
        GRUB_CMDLINE_LINUX_DEFAULT="resume=/dev/mapper/ubuntu--vg-swaplv"
        ```
  - add resume option
    - `sudo nano /etc/initramfs-tools/conf.d/resume`
      - ```
        RESUME=/dev/mapper/ubuntu--vg-swaplv
        ```
  - `sudo update-initramfs -k all -u && sudo update-grub`
  - `reboot`
  - test hibernation with `sudo systemctl hibernate`, at this point it should work

## continue setting up the hibernation

- verify that the BIOS is set up correctly and S4 is present
  - `sudo dmesg | grep "(supports"`
    - ACPI: PM: (supports S0 **S4** S5)
- `sudo nano /etc/systemd/logind.conf`
  - ```
    HandlePowerKey=hibernate
    HandlePowerKeyLongPress=poweroff
    HandleLidSwitch=hibernate
    HandleLidSwitchExternalPower=hibernate
    HandleLidSwitchDocked=hibernate
    ```
- optionally, enable automatic power profile switching, original code from https://kobusvs.co.za/blog/power-profile-switching/

  - `sudo apt install inotify-tools`

  - `nano /home/$USER/powerswitch.sh`

    - ```
      #! /bin/bash


      BAT=$(echo /sys/class/power_supply/BAT*)
        BAT_STATUS="$BAT/status"
      BAT_CAP="$BAT/capacity"
      LOW_BAT_PERCENT=20

      AC_PROFILE="balanced"
      BAT_PROFILE="power-saver"
      LOW_BAT_PROFILE="power-saver"

      # wait a while if needed

      [[-z $STARTUP_WAIT]] || sleep "$STARTUP_WAIT"

      # start the monitor loop

      prev=0

      while true; do # read the current state
      if [[$(cat "$BAT_STATUS") == "Discharging"]]; then
      if [[$(cat "$BAT_CAP") -gt $LOW_BAT_PERCENT]]; then
      profile=$BAT_PROFILE
                else
                    profile=$LOW_BAT_PROFILE
      fi
      else
      profile=$AC_PROFILE
      fi

      # set the new profile
      if [[ $prev != "$profile" ]]; then
          echo setting power profile to $profile
          powerprofilesctl set $profile
      fi

      prev=$profile

      # wait for the next power change event
      inotifywait -qq "$BAT_STATUS" "$BAT_CAP"

      done

      ```

    - `sudo chmod u+x /home/$USER/powerswitch.sh`
    - `nano /home/$USER/.config/systemd/user/power-monitor.service`

      - ```
        [Unit]
        Description=power profile switch
        After=network.target

        [Service]
        Type=simple
        Environment=STARTUP_WAIT=30s
        ExecStart=/home/xxx/powerswitch.sh

        [Install]
        WantedBy=default.target
        ```

    - replace xxx with the actual username
      - `sed -i "s/xxx/$(whoami)/g" /home/$USER/.config/systemd/user/power-monitor.service`
    - `sudo systemctl daemon-reload`
    - `chmod 644 /home/$USER/.config/systemd/user/power-monitor.service`
    - `sudo loginctl enable-linger $USER`
      - disable with `sudo loginctl disable-linger $USER`
      - status: `ls /var/lib/systemd/linger`
    - `systemctl --user enable --now power-monitor.service`
    - `systemctl --user status power-monitor.service`
    - `journalctl --user-unit power-monitor.service -f` to get live status updates

## continue to set up QEMU/KVM

- `nano /etc/modules`
  - ```
    vfio
    vfio_pci
    ```
- `sudo update-initramfs -u -k all`
- `reboot` if there were any problems with the discrete GPU and/or hybrid mode before and it was disabled, it must be enabled back at this point
- `lshw -c video`
- `lspci -nn | grep 'NVIDIA'` note the address on the PCIe bus
- `lspci -n -v -s 32:00.0` note the ID
- `nano /etc/modprobe.d/vfio.conf` fill the ID from above

  - ```
    softdep nouveau pre: vfio vfio-pci
    softdep nvidia pre: vfio vfio-pci
    options vfio-pci ids=10de:28a1
    ```

- `sudo update-initramfs -u -k all`
- `sudo nano /etc/default/grub`
  - ```
    GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt resume=/dev/mapper/ubuntu--vg-swaplv vfio-pci.ids=10de:28a1"
    ```
- `sudo update-grub`
- `reboot`
- `lspci -n -v -s 32:00.0`
  - verify it reports: Kernel driver in use: vfio-pci
- `sudo dmesg | grep -e DMAR -e IOMMU` - all must be present
- `sudo apt install cpu-checker lm-sensors gufw nmon timeshift gnome-tweaks`
  - `kvm-ok`
    - must report that KVM acceleration can be used
  - `sensors` verify that sensors have readings
  - `gufw` enable firewall and set up rules
  - `timeshift` create initial system backup to another volume
  - `deja-dup` create incremental backups later
  - `gnome-tweaks` and set font scale to 1.5 for retina displays
  - open Settings app > Power, turn automatic brightness off
  - `lsusb -v` check recognized USB devices
  - `lspci -nnk` check loaded drivers

## install QEMU

- `sudo apt install qemu-system qemu-kvm virt-manager bridge-utils`
- `sudo useradd -g $USER libvirt && sudo useradd -g $USER kvm && sudo useradd -g $USER libvirt-kvm`
- `reboot`
- import old images and xml, put xml to /etc/libvirt/qemu/
  - if making a new VM disk store, set permissions to the directory `sudo chown libvirt-qemu:kvm ~/images -R && sudo chmod 774 ~/images -R && sudo chmod g+s ~/images`
- run `virt-manager` and enable XML editing in the preferences

## set up VMs

### Ubuntu 22.04 server VM guest

- create VM, x86_64, manually add disk, "customize configuration before install", set bus to SCSI, set discard to unmap, add PCI host device on the address 32:00.0 from before (the NVIDIA GPU).
- regular install, reboot, shutdown
- edit the XML and verify that disk discard='unmap' detect_zeroes='unmap' bus=scsi. set SCSI controller 0 settings to type='scsi' model='virtio-scsi'. all this is to enable TRIM in the guest working correctly and actually having the effect.
- start the VM:
  - `fstrim -a -v`
  - `sudo apt update`
  - `sudo apt install qemu-guest-agent`
  - `sudo systemctl enable --now qemu-guest-agent`
  - `sudo apt install spice-vdagent`
  - `reboot`
- install NVIDIA drivers WITHOUT CUDA

  - `sudo apt install ubuntu-drivers-common` should be already installed
  - `sudo ubuntu-drivers devices`
  - `sudo apt install nvidia-driver-545`
  - `reboot`
  - `nvidia-smi --query-gpu=compute_cap --format=csv`
  - `modinfo nvidia`
  - `nvidia-smi`

- install conda, Pytorch with CUDA

  - `mkdir ~/miniconda3`
  - `wget https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/Miniconda3-py311_24.1.2-0-Linux-x86_64.sh -O ~/miniconda.sh`
  - `bash ~/miniconda.sh -b -u -p ~/miniconda3`
  - `~/miniconda3/bin/conda init bash`
  - `. .bashrc` it's the same as `source .bashrc`, to load the variables into the current session
  - `conda config --set show_channel_urls yes`
  - `nano ~/.condarc` and replace it with content from https://mirrors.tuna.tsinghua.edu.cn/help/anaconda/
  - `pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple`
  - `conda install jupyterlab nb_conda_kernels ipywidgets -c conda-forge -y`
  - `conda create --name pytorch_env python=3.10 ipykernel -c conda-forge -y`
  - `conda activate pytorch_env`
  - `conda install pytorch torchvision torchaudio pytorch-cuda=11.8 -c pytorch -c nvidia -y`
  - test it with `python -c "import torch; print(torch.__version__); out = torch.fft.rfft(torch.randn(1000).cuda()); print(out.sum()); print(torch.cuda.device_count()); print(torch.version.cuda) ; print(torch.backends.cudnn.version()); print(torch.cuda.get_arch_list())"`

### Ubuntu 22.04 desktop VM guest

all of the above, plus

- stop the VM, edit video QXL part of the XML to set vgamem to 65536, otherwise there will be freezes on high resolutions
- disable all display scaling in VM settings and in guest display options, install gnome-tweaks and set font scale to 1.5

## sparse volumes problem

- QEMU disk images are sparse (thin-provisioning), and will display full allocated space (`ls -lh .`) instead of actual space on disk (`du -sh .`). it is possible to compress the image to save space and at the same time make displayed space reflect actual volume size (although compressed). https://forums.unraid.net/topic/150854-trim-with-linux-guest-for-minimal-img-file/
  - `mv ubuntu22.04.qcow2 ubuntu22.04-old.qcow2`
  - `qemu-img convert -c -O qcow2 ubuntu22.04-old.qcow2 ubuntu22.04.qcow2`
  - `sudo chown libvirt-qemu:kvm ubuntu22.04.qcow2`
  - `rm ubuntu22.04-old.qcow2`
