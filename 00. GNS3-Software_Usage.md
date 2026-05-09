# GNS3 Networking Lab VM — Build Guide
*Debian 13.4 XFCE · VirtualBox 7.2.8 · GNS3 3.0.6 · May 2026*

---

## Confirmed System Requirements

| Item | Value |
|---|---|
| Host OS | Windows 10 |
| CPU | Intel (VT-x capable) |
| Hyper-V | Disabled |
| VM Desktop | XFCE (lightweight, optimal for labs) |

## Software Versions

| Software | Version | Notes |
|---|---|---|
| VirtualBox | 7.2.8 | Latest stable — April 21, 2026 |
| VirtualBox Extension Pack | 7.2.8 | Must match exactly |
| Debian Live XFCE ISO | 13.4 "Trixie" — XFCE 4.20 | Latest stable — March 14, 2026 |
| GNS3 | 2.2.58.1 / 3.0.6 | Installed via pip |
| QEMU/KVM | via apt | Installed inside Debian |
| Docker CE | via apt | Official Docker repo |
| Wireshark | 4.4.14 | via apt |

---

## Step 1 — VirtualBox VM Creation

### New VM Wizard Settings

| Field | Value |
|---|---|
| Name | `Debian_GNS3` |
| ISO Image | debian-live-13.4.0-amd64-xfce.iso |
| Type | Linux |
| Version | Debian (64-bit) |
| Skip Unattended Install | **Must be checked** |

### Hardware

| Setting | Value |
|---|---|
| RAM | 8192 MB (4096 MB minimum) |
| CPUs | 4 (min 2) |
| EFI | Unchecked |

### Virtual Hard Disk

| Setting | Value |
|---|---|
| Size | 80 GB |
| Pre-allocate | No (dynamic) |

### Extra Settings (after wizard, before first boot)

| Tab | Setting | Value |
|---|---|---|
| System → Processor | PAE/NX | Enabled |
| System → Processor | Nested VT-x/AMD-V | Enabled |
| System → Acceleration | Paravirtualization | KVM |
| System → Acceleration | VT-x/AMD-V | Enabled |
| System → Acceleration | Nested Paging | Enabled |
| Display → Screen | Video Memory | 128 MB |
| Display → Screen | Graphics Controller | VMSVGA |
| Display → Screen | 3D Acceleration | Enabled |
| Network → Adapter 1 | Type | NAT |
| Network → Adapter 2 | Type | Host-Only Adapter |
| USB | Controller | USB 3.0 (xHCI) |

### Enable Nested Virtualization via CMD (as Administrator)

The "Nested VT-x/AMD-V" checkbox is greyed out in the GUI — set it via command line with the VM powered off:

```cmd
"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" modifyvm "Debian_GNS3" --nested-hw-virt on
```

Verify:

```cmd
"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" showvminfo "Debian_GNS3" | findstr -i "nested"
```

Expected: `Nested VT-x/AMD-V: enabled`

---

## Step 2 — Debian 13.4 XFCE Installation

### Calamares Installer Settings

| Screen | Value |
|---|---|
| Language | English (or preferred) |
| Timezone | Your region |
| Keyboard | Your layout |
| Partitions | Erase disk (full 80 GB virtual disk) |
| Hostname | `gns3lab` |
| User | Create username + password |
| Admin password | Same as user password (check the box) |

Click "Restart Now" when Calamares finishes, press Enter when asked to remove the installation medium. The VM reboots into a fresh Debian XFCE desktop.

---

## Step 3 — VirtualBox Guest Additions

```bash
# Install dependencies
sudo apt update && sudo apt install -y build-essential dkms linux-headers-$(uname -r)

# Mount Guest Additions ISO via VirtualBox menu:
# Devices → Insert Guest Additions CD image...

# Run installer
sudo mkdir -p /mnt/cdrom
sudo mount /dev/cdrom /mnt/cdrom
sudo /mnt/cdrom/VBoxLinuxAdditions.run

# Reboot
sudo reboot
```

After reboot, set in VirtualBox:
- Shared Clipboard: **Bidirectional**
- Drag and Drop: **Bidirectional**

---

## Step 4 — System Update & Essential Base Tools

```bash
sudo apt update && sudo apt full-upgrade -y && sudo apt install -y \
  curl wget git nano vim htop net-tools \
  apt-transport-https ca-certificates gnupg lsb-release \
  bash-completion openssh-server \
  bridge-utils uml-utilities \
  iptables nftables \
  python3 python3-pip python3-venv \
  && sudo apt autoremove -y && sudo apt autoclean
```

> Note: `software-properties-common` is Ubuntu-only and not available in Debian 13 — skip it.

### SSH

```bash
sudo apt install -y openssh-server
sudo systemctl enable ssh && sudo systemctl start ssh

# Verify listening on port 22
ss -tlnp | grep 22
```

| Package | Purpose |
|---|---|
| curl, wget | Downloading files & repos |
| git | Version control |
| nano, vim | Text editors |
| htop | System monitor |
| net-tools | ifconfig, netstat etc. |
| apt-transport-https, ca-certificates, gnupg | Secure repo access |
| lsb-release | Distro info (needed by Docker/GNS3 scripts) |
| bash-completion | Tab completion in terminal |
| openssh-server | Remote SSH access |
| bridge-utils | Network bridging for GNS3 |
| iptables, nftables | Firewall/network rules |
| python3, pip, venv | Python runtime for GNS3 |

---

## Step 5 — KVM + QEMU + libvirt

```bash
sudo apt install -y \
  qemu-kvm qemu-utils \
  libvirt-daemon-system libvirt-clients \
  virtinst virt-manager \
  ovmf \
  libguestfs-tools \
  && sudo systemctl enable libvirtd \
  && sudo systemctl start libvirtd
```

### Verify KVM

```bash
ls -la /dev/kvm
# Expected: crw-rw----+ 1 root kvm ...

lsmod | grep kvm
# Expected: kvm_intel and kvm modules loaded
```

> Note: `kvm-ok` (cpu-checker) is not available in Debian 13 — use the above commands instead.

---

## Step 6 — Docker CE

### Add GPG Key & Repo

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian trixie stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install Docker

```bash
sudo apt update && sudo apt install -y \
  docker-ce docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

sudo systemctl enable docker && sudo systemctl start docker
sudo docker run hello-world
```

---

## Step 7 — Wireshark

```bash
sudo apt install -y wireshark
# Select YES when prompted for non-superuser packet capture

sudo usermod -aG wireshark $USER
wireshark --version | head -1
```

> Note: `wireshark-qt` is not a separate package in Debian 13 — the Qt GUI is bundled inside `wireshark` directly.

---

## Step 8 — GNS3

### Dependencies

```bash
sudo apt install -y \
  python3-pip python3-pyqt5 python3-pyqt5.qtsvg \
  python3-pyqt5.qtwebsockets python3-pyqt5.qtwebengine \
  python3-dev libffi-dev libssl-dev \
  xterm putty
```

### Build dynamips from source

`dynamips` and `vpcs` are not in Debian 13 repos — build from source:

```bash
sudo apt install -y cmake libpcap-dev libelf-dev git
cd /tmp
git clone https://github.com/GNS3/dynamips.git
cd dynamips && mkdir build && cd build
cmake ..
sudo make install
cd ~
```

### Build vpcs from source

```bash
sudo apt install -y libreadline-dev
cd /tmp
git clone https://github.com/GNS3/vpcs.git
cd vpcs/src
./mk.sh
sudo cp vpcs /usr/local/bin/vpcs
sudo chmod +x /usr/local/bin/vpcs
cd ~
```

### Install GNS3 via pip

```bash
sudo pip3 install gns3-server gns3-gui --break-system-packages --ignore-installed psutil
```

### Fix PyQt6 (GNS3 3.0.x requires PyQt6, not PyQt5)

```bash
sudo pip3 install PyQt6 PyQt6-Qt6 PyQt6-sip --break-system-packages --ignore-installed
```

### Fix sip compatibility shim

```bash
# Verify PyQt5.sip works
python3 -c "from PyQt5 import sip; print(sip.SIP_VERSION_STR)"

# Create compatibility shim
echo "from PyQt5.sip import *
from PyQt5 import sip
import sys
sys.modules['sip'] = sip" | sudo tee /usr/local/lib/python3.13/dist-packages/sip.py > /dev/null

# Verify
python3 -c "import sip; print(sip.SIP_VERSION_STR)"
gns3 --version
```

### Key Notes for GNS3 3.0.6 on Debian 13

- Requires PyQt6 (not PyQt5)
- sip 6.x no longer standalone — needs compatibility shim
- Install method: pip3 with `--break-system-packages`

---

## Step 9 — Remote Access Tools

```bash
sudo apt install -y \
  xrdp \
  tigervnc-standalone-server \
  tigervnc-common \
  novnc \
  remmina \
  remmina-plugin-rdp \
  remmina-plugin-vnc

sudo systemctl enable xrdp
sudo systemctl start xrdp
```

| Tool | Purpose |
|---|---|
| xrdp | RDP server — students connect via Windows Remote Desktop |
| tigervnc | VNC server — alternative remote access |
| novnc | Browser-based VNC viewer (no client needed) |
| remmina | GUI remote desktop client (RDP/VNC/SSH) |

---

## Step 10 — User Groups & Permissions

```bash
sudo usermod -aG kvm,libvirt,libvirt-qemu,docker,wireshark,xrdp,ssl-cert $USER

# Reboot to activate all group memberships
sudo reboot
```

After reboot, verify:

```bash
groups
```

| Group | Purpose |
|---|---|
| kvm | KVM virtualization access |
| libvirt | libvirt daemon access |
| libvirt-qemu | QEMU/libvirt access |
| docker | Docker without sudo |
| wireshark | Packet capture without sudo |
| xrdp | RDP remote access |
| ssl-cert | SSL certificates for xrdp |

---

## Step 11 — Network Bridge

```bash
# Check if virbr0 exists (libvirt usually auto-creates it)
ip addr show virbr0 2>/dev/null && echo "EXISTS" || echo "NOT FOUND"
```

If NOT FOUND:

```bash
sudo apt install -y bridge-utils
sudo nano /etc/network/interfaces
# Add at end of file:
# auto virbr0
# iface virbr0 inet static
#     address 192.168.122.1
#     netmask 255.255.255.0
#     bridge_ports none
#     bridge_stp off
#     bridge_fd 0
sudo systemctl restart networking
```

Expected: `virbr0` at `192.168.122.1/24` (NO-CARRIER is normal — activates when VMs connect).

---

## Step 12 — GNS3 First Launch & Configuration

```bash
gns3 &
```

### Setup Wizard Settings

| Screen | Value |
|---|---|
| Run mode | "Run applications on my local computer" |
| Host | 127.0.0.1 |
| Port | 3080 |
| Server path | /usr/local/bin/gns3server |

### Preferences Verification

| Section | Expected Path |
|---|---|
| Server | 127.0.0.1:3080 / /usr/local/bin/gns3server |
| QEMU | /usr/bin/qemu-system-x86_64 |
| Dynamips | /usr/local/bin/dynamips |
| Docker | Available |

---

## Step 13 — Desktop Shortcuts

```bash
# GNS3
cat > ~/Desktop/GNS3.desktop << 'EOF'
[Desktop Entry]
Name=GNS3
Comment=Network Simulation Platform
Exec=gns3
Icon=gns3
Terminal=false
Type=Application
Categories=Network;
EOF

# Wireshark
cat > ~/Desktop/Wireshark.desktop << 'EOF'
[Desktop Entry]
Name=Wireshark
Comment=Network Protocol Analyzer
Exec=wireshark
Icon=wireshark
Terminal=false
Type=Application
Categories=Network;
EOF

# Terminal
cat > ~/Desktop/Terminal.desktop << 'EOF'
[Desktop Entry]
Name=Terminal
Comment=Open Terminal
Exec=xfce4-terminal
Icon=utilities-terminal
Terminal=false
Type=Application
Categories=System;
EOF

chmod +x ~/Desktop/GNS3.desktop
chmod +x ~/Desktop/Wireshark.desktop
chmod +x ~/Desktop/Terminal.desktop
```

---

## Phase A — Network & Terminal Tools

```bash
sudo apt install -y \
  nmap zenmap tcpdump iperf3 \
  ncat netcat-traditional traceroute \
  mtr mtr-tiny arp-scan masscan \
  terminator vim-gtk3 fzf ripgrep bat \
  ranger mc htop iotop iftop nethogs bmon \
  bind9-dnsutils whois sslscan netdiscover
```

```bash
# nikto (not in Debian 13 repos — install from GitHub)
sudo apt install -y libjson-perl libxml-writer-perl libnet-http-perl libnet-ssleay-perl
cd /opt
sudo git clone https://github.com/sullo/nikto.git
sudo ln -s /opt/nikto/program/nikto.pl /usr/local/bin/nikto
```

```bash
# bat symlink (Debian names it batcat)
sudo ln -s /usr/bin/batcat /usr/local/bin/bat
echo "alias bat='batcat'" >> ~/.bashrc
```

```bash
# fzf + rg + vim integration
cat >> ~/.bashrc << 'EOF'
export FZF_DEFAULT_COMMAND='rg --files --hidden --follow --glob "!.git"'
export FZF_DEFAULT_OPTS='--height 40% --layout=reverse --border'
[ -f /usr/share/doc/fzf/examples/key-bindings.bash ] && \
  source /usr/share/doc/fzf/examples/key-bindings.bash
[ -f /usr/share/bash-completion/completions/fzf ] && \
  source /usr/share/bash-completion/completions/fzf
EOF

cat >> ~/.vimrc << 'EOF'
if executable('rg')
  set grepprg=rg\ --vimgrep\ --smart-case
  set grepformat=%f:%l:%c:%m
endif
EOF

source ~/.bashrc
```

| Tool | Version | Binary |
|---|---|---|
| fzf | 0.60.3 | /usr/bin/fzf |
| ripgrep | 14.1.1 | /usr/bin/rg |
| vim-gtk3 | 9.1.1230 | /usr/bin/vim.gtk3 |
| bat | 0.25.0 | /usr/bin/batcat |
| nikto | 2.6.0 | /usr/local/bin/nikto |

---

## Phase B — File Preview & Multimedia

```bash
sudo apt install -y \
  tumbler ffmpegthumbnailer \
  libreoffice libreoffice-gtk3 \
  vlc vlc-bin vlc-plugin-base vlc-plugin-video-output \
  evince okular eog \
  file-roller thunar-archive-plugin thunar-media-tags-plugin thunar-volman \
  gthumb mediainfo-gui \
  fonts-liberation fonts-dejavu \
  unrar-free p7zip-full
```

| Tool | Version |
|---|---|
| LibreOffice | 25.2.3.2 |
| VLC | 3.0.23 |
| Evince | 48.1 |

> Note: `unrar` is not available in Debian 13 — use `unrar-free`. `p7zip-full` covers all archive needs.

---

## Phase C — GUI Management Tools

### Cockpit

```bash
sudo apt install -y cockpit cockpit-networkmanager \
  cockpit-storaged cockpit-packagekit cockpit-doc cockpit-pcp
sudo systemctl enable cockpit.socket
sudo systemctl start cockpit.socket
```

Access: https://localhost:9090

### virt-manager

```bash
sudo apt install -y virt-manager
```

### Webmin

```bash
# Install from .deb (repo uses SHA1, rejected by Debian 13 security policy)
cd /tmp
curl -L https://github.com/webmin/webmin/releases/download/2.630/webmin_2.630_all.deb \
  -o webmin_2.630_all.deb
sudo apt install -y ./webmin_2.630_all.deb
sudo systemctl enable webmin && sudo systemctl start webmin
```

Access: https://localhost:10000

### Portainer CE

```bash
sudo docker volume create portainer_data
sudo docker run -d \
  -p 8000:8000 -p 9443:9443 \
  --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Access: https://localhost:9443

| Tool | Version | Access |
|---|---|---|
| Cockpit | 337 | https://localhost:9090 |
| virt-manager | 5.0.0 | Desktop app |
| Portainer CE | latest | https://localhost:9443 |
| Webmin | 2.630 | https://localhost:10000 |

---

## Phase D — Security Tools

```bash
sudo apt install -y \
  clamav clamav-daemon clamtk \
  rkhunter chkrootkit \
  aide aide-common \
  fail2ban \
  ufw gufw \
  apparmor apparmor-utils \
  apparmor-profiles apparmor-profiles-extra \
  auditd audispd-plugins \
  logwatch \
  libpam-pwquality \
  bleachbit acct needrestart
```

```bash
# ClamAV database update
sudo systemctl stop clamav-freshclam
sudo freshclam
sudo systemctl start clamav-freshclam
sudo systemctl enable clamav-daemon && sudo systemctl start clamav-daemon
```

```bash
# Lynis (not in Debian 13 repos)
cd /tmp
curl -L https://downloads.cisofy.com/lynis/lynis-3.1.4.tar.gz -o lynis.tar.gz
sudo tar -xzf lynis.tar.gz -C /opt
sudo ln -s /opt/lynis/lynis /usr/local/bin/lynis
```

| Tool | Version | Status |
|---|---|---|
| ClamAV | 27995 sigs | running |
| rkhunter | 1.4.6 | installed |
| chkrootkit | — | installed |
| aide | 0.19.1 | installed |
| lynis | 3.1.4 | installed |
| fail2ban | — | running |
| ufw + gufw | — | installed |
| apparmor | — | profiles loaded |
| auditd | — | running |

---

## Phase E — System Hardening

### E1 — UFW Firewall

```bash
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp    comment 'SSH'
sudo ufw allow 3080/tcp  comment 'GNS3'
sudo ufw allow 3389/tcp  comment 'xRDP'
sudo ufw allow 5900/tcp  comment 'VNC'
sudo ufw allow 9090/tcp  comment 'Cockpit'
sudo ufw allow 9443/tcp  comment 'Portainer'
sudo ufw allow 10000/tcp comment 'Webmin'
sudo ufw allow 8000/tcp  comment 'Portainer-edge'
sudo ufw allow 19999/tcp comment 'Netdata'
sudo ufw --force enable
sudo ufw status verbose
```

### E2 — SSH Hardening

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

sudo tee /etc/ssh/sshd_config.d/hardening.conf > /dev/null << 'EOF'
PermitRootLogin no
PasswordAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
MaxAuthTries 3
MaxSessions 5
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
AllowTcpForwarding yes
PermitEmptyPasswords no
Protocol 2
Banner /etc/ssh/banner
EOF

sudo tee /etc/ssh/banner > /dev/null << 'EOF'
***************************************************************************
*  AUTHORIZED ACCESS ONLY — CYBERSECURITY TRAINING LAB                   *
*  This system is for authorized students only.                           *
*  All activity is monitored and logged.                                  *
*  Unauthorized access is strictly prohibited and will be prosecuted.    *
***************************************************************************
EOF

sudo systemctl restart ssh
```

| Setting | Value | Reason |
|---|---|---|
| PermitRootLogin | no | Prevent root SSH access |
| PasswordAuthentication | yes | Students need password login |
| MaxAuthTries | 3 | Limit brute force attempts |
| LoginGraceTime | 30s | Reduce exposure window |
| PermitEmptyPasswords | no | Prevent blank password login |
| Protocol | 2 | Only use SSHv2 |
| Banner | warning text | Legal notice for students |

### E3 — Fail2ban

```bash
sudo tee /etc/fail2ban/jail.local > /dev/null << 'EOF'
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 3
backend = systemd
ignoreip = 127.0.0.1/8 ::1 192.168.122.0/24

[sshd]
enabled  = true
port     = ssh
maxretry = 3
bantime  = 3600
EOF

sudo systemctl restart fail2ban
sudo fail2ban-client status
```

### E4 — Kernel Hardening (sysctl)

```bash
sudo tee /etc/sysctl.d/99-hardening.conf > /dev/null << 'EOF'
net.ipv4.conf.all.rp_filter = 1
net.ipv4.tcp_syncookies = 1
kernel.randomize_va_space = 2
kernel.yama.ptrace_scope = 1
fs.suid_dumpable = 0
kernel.kptr_restrict = 2
net.ipv4.ip_forward = 1
EOF
sudo sysctl -p /etc/sysctl.d/99-hardening.conf
```

> `ip_forward` is kept ON — required for GNS3, KVM, and Docker.

| Setting | Value | Purpose |
|---|---|---|
| rp_filter | 1 | IP spoofing protection |
| tcp_syncookies | 1 | SYN flood protection |
| randomize_va_space | 2 | Full ASLR |
| ptrace_scope | 1 | Restrict process injection |
| suid_dumpable | 0 | Disable core dumps |
| kptr_restrict | 2 | Hide kernel pointers |
| ip_forward | 1 | KEPT ON for GNS3/KVM/Docker |

### E5 — AppArmor + Password Policy + Auto-Updates

```bash
# AppArmor enforce mode
sudo aa-enforce /etc/apparmor.d/* 2>/dev/null
sudo systemctl restart apparmor

# Password policy
sudo tee /etc/security/pwquality.conf > /dev/null << 'EOF'
minlen = 12
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
maxrepeat = 3
gecoscheck = 1
EOF

# Account lockout
sudo tee /etc/security/faillock.conf > /dev/null << 'EOF'
deny = 5
unlock_time = 600
fail_interval = 900
EOF

# Auto security updates
sudo apt install -y unattended-upgrades apt-listchanges
```

| Setting | Value | Purpose |
|---|---|---|
| minlen | 12 | Minimum 12 characters |
| dcredit | -1 | At least 1 digit |
| ucredit | -1 | At least 1 uppercase |
| lcredit | -1 | At least 1 lowercase |
| ocredit | -1 | At least 1 special char |
| deny | 5 | Lock after 5 failed attempts |
| unlock_time | 600 | Unlock after 10 minutes |

### E6 — AIDE + Auditd

```bash
# Initialize AIDE database
sudo aideinit
# Answer Y when prompted
sudo cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db

# Auditd
sudo systemctl restart auditd
sudo auditctl -l | wc -l
```

### E7 — SUID Audit & Final Hardening

```bash
# SUID/SGID audit
sudo find / -perm -4000 -type f 2>/dev/null | sort

# Disable unnecessary services
for service in avahi-daemon cups bluetooth ModemManager; do
  sudo systemctl disable --now $service 2>/dev/null
done

# Secure shared memory
echo "tmpfs /run/shm tmpfs defaults,noexec,nosuid 0 0" | \
  sudo tee -a /etc/fstab > /dev/null

# Weekly security scans via cron
sudo tee /etc/cron.weekly/security-scan > /dev/null << 'EOF'
#!/bin/bash
rkhunter --check --skip-keypress --report-warnings-only
chkrootkit 2>/dev/null
lynis audit system --quiet
EOF
sudo chmod +x /etc/cron.weekly/security-scan

# rkhunter database update
sudo rkhunter --update && sudo rkhunter --propupd
```

---

## Phase F — Final Verification

```bash
echo "=== GNS3 Server ===" && gns3server --version
echo "=== GNS3 GUI ===" && gns3 --version
echo "=== KVM ===" && ls -la /dev/kvm
echo "=== Docker ===" && sudo docker ps
echo "=== Wireshark ===" && wireshark --version | head -1
echo "=== SSH ===" && sudo systemctl status ssh | grep Active
echo "=== libvirtd ===" && sudo systemctl status libvirtd | grep Active
echo "=== xrdp ===" && sudo systemctl status xrdp | grep Active
echo "=== cockpit ===" && sudo systemctl status cockpit.socket | grep Active
echo "=== webmin ===" && sudo systemctl status webmin | grep Active
echo "=== fail2ban ===" && sudo systemctl status fail2ban | grep Active
echo "=== ufw ===" && sudo ufw status | head -2
echo "=== apparmor ===" && sudo aa-status | grep "profiles are in enforce mode"
echo "=== auditd ===" && sudo systemctl status auditd | grep Active
echo "=== clamav ===" && sudo systemctl status clamav-daemon | grep Active
```

### Expected Results

| Service | Version | Status |
|---|---|---|
| GNS3 Server | 3.0.6 | running |
| GNS3 GUI | 3.0.6 | running |
| KVM /dev/kvm | — | exists |
| Docker + Portainer | — | running |
| Wireshark | 4.4.14 | installed |
| SSH | — | active (running) |
| libvirtd | — | active (running) |
| xrdp | — | active (running) |
| Cockpit | — | active (listening) |
| Webmin | 2.630 | active (running) |
| fail2ban | — | active (running) |
| UFW | — | active |
| AppArmor | — | 165 profiles enforce |
| auditd | — | active (running) |
| ClamAV | — | active (running) |

---

## GNS3 Critical Fixes

### uBridge & virbr0

```bash
# Build uBridge from source
sudo apt install -y libpcap-dev cmake
cd /tmp
git clone https://github.com/GNS3/ubridge.git
cd ubridge
make
sudo make install
which ubridge

# libvirt NAT network
sudo systemctl enable libvirtd --now
sudo virsh net-start default 2>/dev/null
sudo virsh net-autostart default
ip addr show virbr0 | grep inet

# uBridge capabilities
sudo apt install -y libcap2-bin
sudo setcap cap_net_admin,cap_net_raw=eip /usr/local/bin/ubridge
sudo setcap cap_net_admin,cap_net_raw=eip /usr/local/bin/dynamips
sudo usermod -aG netdev $USER

# Add /usr/sbin to PATH permanently
echo 'export PATH=$PATH:/usr/sbin' >> ~/.bashrc
source ~/.bashrc
```

### Config Paths (fix via Python)

```python
config['VPCS']['vpcs_path'] = '/usr/local/bin/vpcs'
config['Dynamips']['dynamips_path'] = '/usr/local/bin/dynamips'
config['Controller']['ubridge_path'] = '/usr/local/bin/ubridge'
```

### Verified Capabilities

| Binary | Capabilities |
|---|---|
| /usr/local/bin/ubridge | cap_net_admin,cap_net_raw=eip |
| /usr/local/bin/dynamips | cap_net_admin,cap_net_raw=eip |

### Verified Config Paths

| Setting | Value |
|---|---|
| VPCS path | /usr/local/bin/vpcs |
| Dynamips path | /usr/local/bin/dynamips |
| uBridge path | /usr/local/bin/ubridge |
| NAT interface | virbr0 (192.168.122.1) |
| Terminal command | `xterm -T "%d" -e "telnet %h %p"` |

### VPCS Console Fix

```bash
# telnet must be installed — xterm opens, runs telnet, closes if not found
sudo apt install -y telnet
```

Then in GNS3: Edit → Preferences → General → Console command:
`xterm -T "%d" -e "telnet %h %p"`

### VPCS Internet Access

Connect the VPCS node to a NAT (Built-in) node on the canvas. Inside VPCS:

```
dhcp          # get IP from NAT
ping 8.8.8.8  # test connectivity
```

---

## Desktop Integration

### Ranger + File Previews

```bash
sudo apt install -y \
  ranger w3m w3m-img ueberzug python3-pil \
  ffmpeg highlight atool poppler-utils \
  mediainfo odt2txt catdoc libsixel-bin

ranger --copy-config=all
sed -i 's/set preview_images false/set preview_images true/' ~/.config/ranger/rc.conf
sed -i 's/set preview_images_method w3m/set preview_images_method ueberzug/' ~/.config/ranger/rc.conf
```

| File Type | Preview Tool |
|---|---|
| Images | ueberzug |
| Videos | ffmpeg thumbnails |
| PDFs | poppler-utils |
| Office docs | odt2txt / catdoc |
| Archives | atool |
| Code/text | highlight |
| Media info | mediainfo |

### Conky + XFCE Panel Plugins

```bash
sudo apt install -y \
  conky-all \
  xfce4-cpugraph-plugin \
  xfce4-netload-plugin \
  xfce4-systemload-plugin \
  xfce4-weather-plugin \
  xfce4-taskmanager \
  xfce4-pulseaudio-plugin \
  xfce4-clipman-plugin \
  lm-sensors smartmontools
```

Conky displays: CPU cores, RAM, Swap, Disk, Network, Top 5 processes, Services status, Clock. Position: top-right corner. Update interval: 2 seconds. Autostart: `~/.config/autostart/conky.desktop`.

> Note: Running inside VirtualBox — no CPU temperature sensors are exposed (expected behaviour).

---

## Batch 1 — Modern Monitoring Tools

```bash
sudo apt install -y \
  btop ncdu duf glances nload iotop iftop nethogs nmon sysstat \
  lnav memtester stress-ng ioping gsmartcontrol \
  s-tui bmon wavemon speedtest-cli iperf whois dnstop

sudo apt install -y fastfetch
sudo apt install -y tealdeer && tldr --update

# ctop binary
sudo wget -q https://github.com/bcicen/ctop/releases/download/v0.7.7/ctop-0.7.7-linux-amd64 \
  -O /usr/local/bin/ctop && sudo chmod +x /usr/local/bin/ctop

# bottom (btm)
wget -q https://github.com/ClementTsang/bottom/releases/download/0.10.2/bottom_0.10.2-1_amd64.deb \
  -O /tmp/bottom.deb && sudo apt install -y /tmp/bottom.deb
```

| Tool | Version |
|---|---|
| btop | installed |
| fastfetch | 2.40.4 |
| tealdeer (tldr) | 1.7.2 |
| ctop | 0.7.7 |
| bottom (btm) | 0.10.2 |
| glances | installed |
| s-tui | installed |
| nmon, lnav, ncdu, duf | installed |

## Batch 2 — Network & Modern CLI Tools

```bash
sudo apt install -y bandwhich gping trippy termshark procs du-dust fd-find xh
```

### Netdata

```bash
curl -fsSL https://get.netdata.cloud/kickstart.sh | sudo sh -s -- --non-interactive --stable-channel
sudo systemctl status netdata | grep Active
sudo ufw allow 19999/tcp comment 'Netdata'
```

Access: http://localhost:19999

### kdig (DNS tool — replaces doggo/dog)

```bash
# doggo/dog are unavailable or unreliable on Debian 13
# kdig (from knot-dnsutils) is the replacement
sudo apt install -y knot-dnsutils
echo "alias dog='kdig'" >> ~/.bashrc
echo "alias doggo='kdig'" >> ~/.bashrc
source ~/.bashrc
```

### DNS Final State

| Tool | Status |
|---|---|
| kdig (knot-dnsutils 3.4.6) | installed — aliased as dog/doggo |
| drill (ldnsutils) | installed |
| bandwhich | building via cargo (background) |

---

## Phase G — Final Cleanup & OVA Export

```bash
pkill -f gns3 2>/dev/null
sudo apt clean && sudo apt autoremove -y
history -c && history -w
sudo journalctl --rotate && sudo journalctl --vacuum-time=1s
sudo rm -rf /tmp/* /var/tmp/*
sudo rm -f /var/lib/aide/aide.db.new
sudo dd if=/dev/zero of=/zeroes bs=1M 2>/dev/null; sudo rm -f /zeroes
echo "Cleanup complete — ready for shutdown!"
sudo shutdown -h now
```

> After shutdown — **do not restart the VM**. Export directly as OVA from VirtualBox.

---

## Web Dashboards Summary

| Tool | URL | Purpose |
|---|---|---|
| Netdata | http://localhost:19999 | Real-time performance |
| Cockpit | https://localhost:9090 | System management |
| Portainer | https://localhost:9443 | Docker management |
| Webmin | https://localhost:10000 | Full admin panel |
| Glances | http://localhost:61208 | System monitor (run: `glances -w`) |

---

## Tool Usage Guide

---

### System Monitors

#### btop — All-in-one TUI monitor

```bash
btop
# Keys: q=quit, F2=setup, m=memory, n=network, p=process
```

#### bottom (btm) — Modern htop with graphs

```bash
btm
# Keys: q=quit, Tab=switch panels, f=filter, s=sort
# dd=kill process, c=CPU sort, m=RAM sort
```

#### glances — Monitor with web interface

```bash
glances              # TUI mode
glances -w           # Web server → http://localhost:61208
glances --export csv # Export stats to CSV
```

#### htop — Classic interactive monitor

```bash
htop
# F5=tree view, F6=sort, F9=kill, F10=quit
# u=filter by user, /=search
```

#### nmon — Performance monitor with logging

```bash
nmon                 # Interactive TUI
nmon -f -s 5 -c 60  # Log to file (every 5s, 60 times)
# Keys: c=CPU, m=memory, n=network, d=disk, t=top processes
```

#### s-tui — CPU frequency + stress TUI

```bash
s-tui
# Shows CPU frequency, utilisation, temperature
# Built-in stress test mode
```

#### fastfetch — System info

```bash
fastfetch
fastfetch --logo    # With distro logo
```

---

### Disk Tools

#### ncdu — Interactive disk usage

```bash
ncdu /              # Scan root filesystem
ncdu ~              # Scan home directory
# Keys: d=delete, i=info, q=quit, /=select
```

#### duf — Modern df replacement

```bash
duf                 # Show all mounts
duf /home           # Specific path
duf --only local    # Local filesystems only
```

#### dust — Modern du replacement

```bash
dust                # Current directory usage
dust /var           # Specific directory
dust -n 20          # Top 20 items
```

#### ioping — Disk latency tester

```bash
ioping .            # Test current disk latency
ioping -c 10 /      # 10 requests to root
ioping -R /dev/sda  # Disk seek rate test
```

#### gsmartcontrol — HDD/SSD health GUI

```bash
gsmartcontrol       # Launch GUI
# View SMART data, run tests, check health
```

---

### Network Tools

#### bandwhich — Per-process bandwidth TUI

```bash
sudo bandwhich      # Show bandwidth per process/connection
```

#### gping — Visual ping with graph

```bash
gping 8.8.8.8              # Ping with graph
gping google.com 1.1.1.1   # Multiple hosts
gping --cmd "curl google.com"
```

#### trippy — Modern traceroute TUI

```bash
sudo trip 8.8.8.8
sudo trip --protocol udp google.com
```

#### termshark — Terminal Wireshark TUI

```bash
sudo termshark -i enp0s3           # Capture on interface
sudo termshark -r file.pcap        # Read pcap file
```

#### nload — Real-time bandwidth per interface

```bash
nload               # All interfaces
nload enp0s3        # Specific interface
# Keys: arrows=switch interface, F2=options
```

#### nethogs — Per-process network usage

```bash
sudo nethogs
sudo nethogs enp0s3
```

#### iftop — Real-time connection monitor

```bash
sudo iftop
sudo iftop -i enp0s3 -n  # No DNS resolution
# Keys: n=hostname/IP, p=ports, s=sort, q=quit
```

#### bmon — Bandwidth monitor TUI

```bash
bmon
bmon -p enp0s3
```

#### kdig — DNS lookup tool

```bash
kdig google.com           # Basic DNS lookup
kdig -T google.com        # DNS trace
kdig MX gmail.com         # Mail records
kdig @8.8.8.8 google.com  # Use specific DNS server
kdig -d google.com        # Debug mode
```

Aliased as `dog` and `doggo`.

#### nmtui — NetworkManager TUI

```bash
nmtui
# Edit connections, activate/deactivate, set hostname
```

#### nmap — Network scanner

```bash
nmap 192.168.1.0/24          # Scan subnet
nmap -sV -p 1-65535 host     # Full port scan with versions
nmap -A target               # Aggressive scan (OS detect + scripts)
sudo nmap -sS target         # Stealth SYN scan
```

#### Zenmap — Nmap GUI

```bash
zenmap
# Save profiles, visualise network topology
```

#### netdiscover — ARP network scanner

```bash
sudo netdiscover -r 192.168.1.0/24
sudo netdiscover -i enp0s3
```

#### masscan — Ultra-fast port scanner

```bash
sudo masscan -p 80,443 192.168.0.0/16 --rate=1000
sudo masscan -p 1-65535 target --rate=10000
```

#### xh — Modern HTTP client

```bash
xh GET httpbin.org/get
xh POST httpbin.org/post a=1
xh -j GET api.github.com       # JSON request
xh --headers GET google.com    # Show headers only
```

#### iperf3 — Bandwidth testing

```bash
# Server side:
iperf3 -s
# Client side:
iperf3 -c server_ip
iperf3 -c server_ip -t 30 -P 4  # 30s, 4 parallel streams
```

#### tcpdump — Packet capture

```bash
sudo tcpdump -i enp0s3              # Capture all traffic
sudo tcpdump -i enp0s3 port 80      # HTTP only
sudo tcpdump -w capture.pcap        # Save to file
sudo tcpdump -r capture.pcap        # Read file
```

---

### Process & System Tools

#### procs — Modern ps replacement

```bash
procs
procs nginx         # Filter by name
procs --tree        # Process tree
procs --sort cpu    # Sort by CPU
```

#### fd — Modern find replacement

```bash
fd pattern          # Find files matching pattern
fd -e py            # Find .py files
fd -t f             # Files only
fd passwd /etc
```

#### bat — Modern cat with syntax highlighting

```bash
bat file.py
bat /etc/ssh/sshd_config
bat --paging=never file
```

#### lnav — Log file navigator

```bash
lnav /var/log/syslog
lnav /var/log/auth.log
lnav *.log
# Keys: /, n=next match, e=errors only, i=info
```

#### tldr (tealdeer) — Quick command help

```bash
tldr tar
tldr git
tldr --update
```

#### stress-ng — System stress testing

```bash
stress-ng --cpu 4 --timeout 60s    # CPU stress 60s
stress-ng --vm 2 --vm-bytes 512M   # RAM stress
stress-ng --all 1 --timeout 30s    # All resources
```

#### memtester — RAM fault finder

```bash
sudo memtester 512M 3   # Test 512MB, 3 iterations
```

---

### Security Tools

#### Wireshark — Packet analyzer GUI

```bash
wireshark
wireshark -i enp0s3
# Filters: tcp.port==80, ip.addr==192.168.1.1
```

#### ClamTK — Antivirus GUI

```bash
clamtk
# Scan files, update definitions, schedule scans
```

#### rkhunter — Rootkit scanner

```bash
sudo rkhunter --check
sudo rkhunter --check --skip-keypress  # Non-interactive
sudo rkhunter --propupd                # Update file properties DB
```

#### lynis — Security audit

```bash
sudo lynis audit system
sudo lynis audit system --quick
# Report saved to /var/log/lynis.log
```

#### fail2ban — Brute force protection

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unbanip IP
```

#### GUFW — Firewall GUI

```bash
gufw
# Add/remove rules, enable/disable UFW graphically
```

#### auditd — System audit logs

```bash
sudo aureport --summary
sudo aureport --failed
sudo ausearch -k sudoers
sudo ausearch -ui 1000
```

---

### Container Tools

#### ctop — Docker container monitor TUI

```bash
sudo ctop
# Keys: q=quit, s=stop, r=restart, l=logs, e=exec
```

#### Portainer — Docker GUI

```
Access: https://localhost:9443
# Manage containers, images, volumes, networks
# Deploy stacks, view logs, exec into containers
```

---

### File Managers

#### ranger — Terminal file manager with previews

```bash
ranger
# Keys: h/j/k/l=navigate, Enter=open, q=quit
# yy=copy, dd=cut, pp=paste, :delete=delete
# zh=show hidden, zp=toggle preview
```

#### Midnight Commander (mc) — Classic TUI file manager

```bash
mc
# F5=copy, F6=move, F8=delete, F10=quit
# Tab=switch panel, Insert=select files
```
