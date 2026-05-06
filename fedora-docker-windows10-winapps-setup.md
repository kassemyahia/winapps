# Fedora + Docker + Windows 10 Pro Setup for WinApps

Below is a Fedora + Docker + Windows 10 Pro setup path for WinApps. The important idea is: **Docker is not running Windows apps directly**. WinApps runs a **Windows VM** through QEMU/KVM inside a Docker container, then uses **FreeRDP RemoteApp** to show individual Windows app windows on your Linux desktop. WinApps itself says it runs Windows in Docker/Podman/libvirt, queries installed apps, creates Linux shortcuts, and renders them with FreeRDP. ([GitHub][1])

One warning first: Windows 10 Pro will work technically, but normal Windows 10 support ended on October 14, 2025, unless you are using ESU/LTSC or another supported licensing path. ([Microsoft Support][2])

## 0. What you need

Assume:

```text
Fedora desktop
x86_64 PC
Hardware virtualization enabled in BIOS/UEFI
Docker Engine, not Docker Desktop
Windows 10 Pro / Enterprise / Server, not Home
At least 8 GB host RAM, preferably 16 GB+
At least 80–150 GB free disk space
```

WinApps’ Docker/Podman backend needs GNU/Linux kernel interfaces like KVM for acceptable VM performance, and its docs specifically say Windows Professional, Enterprise, or Server editions are required for RDP applications; Windows Home is not enough. ([GitHub][3])

Check KVM:

```bash
lscpu | grep -i virtualization
ls -l /dev/kvm
```

You want to see something like `VT-x` or `AMD-V`, and `/dev/kvm` should exist. If `/dev/kvm` is missing, enable Intel VT-x / AMD-V / SVM in BIOS. If Fedora itself is inside another VM, you also need nested virtualization.

## 1. Install Docker Engine on Fedora

Docker’s official Fedora instructions currently use Docker’s RPM repository, install `docker-ce`, `docker-ce-cli`, `containerd.io`, Buildx, and the Compose plugin, then start the Docker service. ([Docker Documentation][4])

```bash
sudo dnf remove -y docker \
  docker-client \
  docker-client-latest \
  docker-common \
  docker-latest \
  docker-latest-logrotate \
  docker-logrotate \
  docker-selinux \
  docker-engine-selinux \
  docker-engine

sudo dnf config-manager addrepo --from-repofile https://download.docker.com/linux/fedora/docker-ce.repo

sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl enable --now docker

sudo docker run hello-world
```

WinApps is meant to control Docker from your user session, so you usually want your user to run `docker` without `sudo`. Docker’s docs also warn that membership in the `docker` group effectively grants root-level privileges, so only do this on a machine where that is acceptable. ([Docker Documentation][5])

```bash
sudo groupadd docker 2>/dev/null || true
sudo usermod -aG docker "$USER"
newgrp docker

docker run hello-world
docker compose version
```

Logging out and back in is better than `newgrp docker`, but `newgrp` lets you continue immediately.

## 2. Install WinApps dependencies on Fedora

WinApps lists these Fedora/RHEL dependencies and requires FreeRDP 3 or later. ([GitHub][1])

```bash
sudo dnf install -y curl dialog freerdp git iproute libnotify nmap-ncat
```

Check your FreeRDP binary name:

```bash
command -v xfreerdp3 || command -v xfreerdp
```

## 3. Load the iptables kernel modules for folder sharing

WinApps says the `ip_tables` and `iptable_nat` kernel modules must be loaded for host folder sharing to work. ([GitHub][3])

```bash
lsmod | grep ip_tables
lsmod | grep iptable_nat
```

If either command prints nothing:

```bash
printf "ip_tables\niptable_nat\n" | sudo tee /etc/modules-load.d/iptables.conf
sudo modprobe ip_tables
sudo modprobe iptable_nat
```

Reboot later if folder sharing still fails.

## 4. Get the WinApps files

Keep the repo source in `~/.local/src/winapps`, but put the active Docker config in `~/.config/winapps`.

```bash
mkdir -p ~/.local/src ~/.config/winapps

git clone https://github.com/winapps-org/winapps.git ~/.local/src/winapps

cp ~/.local/src/winapps/compose.yaml ~/.config/winapps/compose.yaml
cp -r ~/.local/src/winapps/oem ~/.config/winapps/oem

cd ~/.config/winapps
```

The `oem` folder matters because the Windows container can run bundled setup scripts during Windows installation. Dockur’s Windows image documents that a folder mounted to `/oem` is copied into `C:\OEM` and `install.bat` is executed during the final automatic installation step. ([GitHub][6])

## 5. Edit `compose.yaml` for Windows 10, RAM, CPU, and disk size

Open the file:

```bash
nano ~/.config/winapps/compose.yaml
```

Do **not** replace the whole file unless you know what you are doing. In the existing `environment:` section, set values like this:

```yaml
environment:
  VERSION: "10"
  RAM_SIZE: "8G"
  CPU_CORES: "4"
  DISK_SIZE: "128G"
  USERNAME: "winapps"
  PASSWORD: "CHANGE_ME_TO_A_REAL_PASSWORD"
```

Meaning:

```text
VERSION: "10"       -> Windows 10 Pro
RAM_SIZE: "8G"      -> VM gets 8 GB RAM
CPU_CORES: "4"      -> VM gets 4 virtual CPU cores
DISK_SIZE: "128G"   -> Windows virtual disk size
USERNAME/PASSWORD   -> Windows account created during install
```

Dockur’s Windows image documents `VERSION: "10"` as Windows 10 Pro, `DISK_SIZE` for increasing the default disk, and `RAM_SIZE` / `CPU_CORES` for VM resources. ([GitHub][6])

### Change where the VM disk is stored

Find the volume line that maps something to `/storage`.

If you want Docker’s default named volume, leave it alone. That stores the Windows disk under Docker’s storage area, usually somewhere like:

```text
/var/lib/docker/volumes/...
```

If you want the Windows VM disk on a specific drive or folder, use a bind mount. For example:

```yaml
volumes:
  - /home/YOUR_LINUX_USERNAME/VMs/winapps-windows:/storage
  - ${HOME}:/shared
  - ./oem:/oem
```

Create the folder first:

```bash
mkdir -p ~/VMs/winapps-windows
```

Dockur’s docs say the `/storage` mount controls the storage location and can be replaced with a desired folder or named volume. ([GitHub][6])

Use a bind mount **before the first boot** if possible. Moving an already-installed VM is possible, but you should stop the container and copy the existing storage carefully.

## 6. Start the Windows 10 VM container

From the config folder:

```bash
cd ~/.config/winapps
docker compose --file ./compose.yaml up -d
```

Watch logs:

```bash
docker compose --file ./compose.yaml logs -f
```

Then open this in your browser:

```text
http://127.0.0.1:8006
```

WinApps’ Docker guide says Windows installation is initiated with `docker compose`, then you complete/access setup through the VNC web viewer at `127.0.0.1:8006`. ([GitHub][3])

The first boot can take a while. It has to download the Windows ISO, create the virtual disk, run the installer, reboot a few times, and apply the OEM setup.

When the Windows desktop appears, log in with the username/password you set in `compose.yaml`.

## 7. Create the WinApps config file

Now create:

```bash
nano ~/.config/winapps/winapps.conf
```

Use this:

```bash
##################################
#   WINAPPS CONFIGURATION FILE   #
##################################

RDP_USER="winapps"
RDP_PASS="CHANGE_ME_TO_THE_SAME_PASSWORD"
RDP_ASKPASS=""
RDP_DOMAIN=""

RDP_IP="127.0.0.1"
VM_NAME="RDPWindows"

WAFLAVOR="docker"

RDP_SCALE="100"
REMOVABLE_MEDIA="/run/media"

RDP_FLAGS="/cert:tofu /sound /microphone +home-drive"
RDP_FLAGS_NON_WINDOWS=""
RDP_FLAGS_WINDOWS=""

DEBUG="true"

AUTOPAUSE="off"
AUTOPAUSE_TIME="300"
```

Lock down the file because it contains your Windows password:

```bash
chmod 600 ~/.config/winapps/winapps.conf
```

WinApps’ sample config uses `RDP_USER`, `RDP_PASS`, `RDP_IP="127.0.0.1"` for Docker/Podman, and `WAFLAVOR="docker"`. ([GitHub][1])

## 8. Test RDP before installing WinApps shortcuts

Run:

```bash
RDP_BIN="$(command -v xfreerdp3 || command -v xfreerdp)"

"$RDP_BIN" /u:winapps /p:'CHANGE_ME_TO_THE_SAME_PASSWORD' /v:127.0.0.1 /cert:tofu /sound +clipboard
```

You should get a full Windows desktop. If RDP fails, fix that before running the WinApps installer.

Common causes:

```text
Wrong Windows password
Windows still installing
Container not running
Port 3389 already in use
FreeRDP certificate mismatch
```

If you get a FreeRDP certificate warning after recreating the VM, remove the old certificate:

```bash
rm -f ~/.config/freerdp/server/127.0.0.1_3389.pem
```

WinApps documents this exact certificate path pattern for stale RDP certificates. ([GitHub][1])

## 9. Install the Windows apps you want

Inside the Windows VM, install whatever Windows apps you want WinApps to expose, for example:

```text
Microsoft Office
Adobe apps
Notepad++
PowerShell tools
Visual Studio
Custom .exe software
```

Use normal Windows installers inside the VM. WinApps later scans Windows for installed applications and creates Linux launchers.

## 10. Run the WinApps installer on Fedora

The official docs run the setup script while Windows is powered on. ([GitHub][1])

Safer inspect-then-run version:

```bash
curl -L https://raw.githubusercontent.com/winapps-org/winapps/main/setup.sh -o /tmp/winapps-setup.sh
less /tmp/winapps-setup.sh
bash /tmp/winapps-setup.sh
```

During the installer:

```text
Choose install
Use the current user unless you want system-wide launchers
Select Docker backend if asked
Let it scan the Windows VM
Choose the apps you want integrated
```

After that, your selected Windows apps should appear in your GNOME/KDE/XFCE launcher like normal Linux apps.

You can also open a full Windows session with:

```bash
winapps windows
```

And run a manual Windows executable:

```bash
winapps manual "notepad.exe"
```

## 11. Everyday control commands

Use these from anywhere:

```bash
docker compose --file ~/.config/winapps/compose.yaml start
docker compose --file ~/.config/winapps/compose.yaml stop
docker compose --file ~/.config/winapps/compose.yaml restart
docker compose --file ~/.config/winapps/compose.yaml pause
docker compose --file ~/.config/winapps/compose.yaml unpause
```

WinApps documents those as the normal subsequent-use Docker commands. ([GitHub][3])

## 12. Change RAM, CPU, or disk size later

For RAM or CPU, edit:

```bash
nano ~/.config/winapps/compose.yaml
```

Change:

```yaml
RAM_SIZE: "12G"
CPU_CORES: "6"
```

Then recreate the container:

```bash
docker compose --file ~/.config/winapps/compose.yaml down
rm -f ~/.config/freerdp/server/127.0.0.1_3389.pem
docker compose --file ~/.config/winapps/compose.yaml up -d
```

WinApps says changes to `compose.yaml` require removing and recreating the container, and this should not affect your data. ([GitHub][3])

For disk size, you can increase:

```yaml
DISK_SIZE: "256G"
```

Then:

```bash
docker compose --file ~/.config/winapps/compose.yaml down
docker compose --file ~/.config/winapps/compose.yaml up -d
```

But inside Windows, the added space may appear as unallocated. Dockur says you must manually extend the Windows partition after increasing the virtual disk. ([GitHub][6])

Do **not** shrink `DISK_SIZE` on an existing Windows VM. If you want a smaller disk, back up your files and make a new VM.

## 13. Add a second virtual disk instead of enlarging C:

Add this to `environment:`:

```yaml
DISK2_SIZE: "64G"
```

And add another storage mount:

```yaml
volumes:
  - /home/YOUR_LINUX_USERNAME/VMs/winapps-windows:/storage
  - /home/YOUR_LINUX_USERNAME/VMs/winapps-disk2:/storage2
  - ${HOME}:/shared
  - ./oem:/oem
```

Then recreate:

```bash
mkdir -p ~/VMs/winapps-disk2

docker compose --file ~/.config/winapps/compose.yaml down
docker compose --file ~/.config/winapps/compose.yaml up -d
```

Dockur documents extra disks with `DISK2_SIZE`, `DISK3_SIZE`, and `/storage2`, `/storage3` mounts. ([GitHub][6])

## 14. Reset everything and start over

This deletes the Windows VM data:

```bash
cd ~/.config/winapps
docker compose down --rmi=all --volumes
```

WinApps documents that command as the “start from scratch” reset path for Docker. ([GitHub][3])

## My recommended starting values

For a normal Fedora laptop/desktop with 16 GB RAM:

```yaml
VERSION: "10"
RAM_SIZE: "8G"
CPU_CORES: "4"
DISK_SIZE: "128G"
```

For a stronger machine with 32 GB RAM:

```yaml
VERSION: "10"
RAM_SIZE: "12G"
CPU_CORES: "6"
DISK_SIZE: "256G"
```

Leave enough resources for Fedora itself. Do not give Windows all CPU cores or almost all RAM, or your Linux desktop will feel frozen.

[1]: https://github.com/winapps-org/winapps "GitHub - winapps-org/winapps: Run Windows apps such as Microsoft Office/Adobe in Linux (Ubuntu/Fedora) and GNOME/KDE as if they were a part of the native OS, including Nautilus integration. Hard fork of https://github.com/Fmstrat/winapps/ · GitHub"
[2]: https://support.microsoft.com/en-us/windows/windows-10-support-has-ended-on-october-14-2025-2ca8b313-1946-43d3-b55c-2b95b107f281?utm_source=chatgpt.com "Windows 10 support has ended on October 14, 2025"
[3]: https://github.com/winapps-org/winapps/blob/main/docs/docker.md "winapps/docs/docker.md at main · winapps-org/winapps · GitHub"
[4]: https://docs.docker.com/engine/install/fedora/ "Install Docker Engine on Fedora | Docker Docs"
[5]: https://docs.docker.com/engine/install/linux-postinstall/ "Linux post-installation steps for Docker Engine | Docker Docs"
[6]: https://github.com/dockur/windows "GitHub - dockur/windows: Windows inside a Docker container. · GitHub"
