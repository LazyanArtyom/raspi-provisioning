# Raspberry Pi Swarm Provisioning

This Ansible project configures Raspberry Pi OS Lite devices for the UAV swarm setup.

It is designed for a fleet of Raspberry Pis where each board may have:

- a different hostname
- a different Wi-Fi/LAN address
- a different LTE static public IP
- a different LTE gateway/network
- different MAVLink endpoint ports

The project configures:

- Hostname per Raspberry Pi
- SSH on ports `22` and `2222`
- NetworkManager + ModemManager for Telit LTE modems
- LTE static public IP access
- LTE source-based policy routing
- Reverse-path filtering disabled for LTE return traffic
- MAVLink tools:
  - `mavlink-router`
  - `MAVProxy`
  - `pymavlink`
- Optional systemd service for `mavlink-router`

The current real Raspberry Pi is configured as:

- Hostname: `drone-pi-01`
- Wi-Fi/LAN SSH address: `192.168.123.49`
- LTE static IP: `178.160.252.170`
- LTE gateway: `178.160.252.169`
- LTE network: `178.160.252.168/30`
- LTE interface: `wwp1s0u1u3i2`
- LTE APN: `internet`

---

## 1. Initial Raspberry Pi OS Lite installation

### 1.1 Download Raspberry Pi Imager

Download and install Raspberry Pi Imager from the official Raspberry Pi website.

Open Raspberry Pi Imager.

### 1.2 Choose OS

Click:

```text
Choose OS
```

Then select:

```text
Raspberry Pi OS (other)
```

Then select:

```text
Raspberry Pi OS Lite 64-bit
```

Use the Lite version because the swarm node does not need a desktop environment. Terminal/server mode is enough.

### 1.3 Choose storage

Insert the SD card into your computer.

Click:

```text
Choose Storage
```

Select the SD card.

Be careful to select the correct device because the selected storage will be erased.

### 1.4 Configure OS customization

Before writing the image, open Raspberry Pi Imager settings / OS customization.

Set a unique hostname for each Raspberry Pi.

This is important because every drone node must be easy to differentiate on the network, in SSH sessions, in logs, and in Ansible inventory.

Example for the first Pi:

```text
Hostname: drone-pi-01
```

Examples for other Pis:

```text
Hostname: drone-pi-02
Hostname: drone-pi-03
Hostname: drone-pi-04
```

Do not give all Pis the same hostname.

Set the username and password.

For the current drone Raspberry Pi setup, use:

```text
Username: admin
Password: adminadmin
```

This keeps all drone nodes consistent during initial provisioning.

> Security note: `adminadmin` is convenient for lab/provisioning work, but for real field deployment it is better to switch to SSH keys and disable password login later.

Enable SSH:

```text
Enable SSH: yes
```

Recommended SSH mode:

```text
Allow public-key authentication
```

If SSH keys are not ready yet, password login is acceptable for the first setup. Later, SSH keys are recommended.

Configure Wi-Fi:

```text
SSID: your Wi-Fi network name
Password: your Wi-Fi password
Country: your country
```

Example:

```text
SSID: iNet_20039
Password: your_wifi_password
Country: AM
```

Set locale/timezone if needed.

Example:

```text
Timezone: Asia/Yerevan
Keyboard: us
```

### 1.5 Write image

Click:

```text
Write
```

Wait until Raspberry Pi Imager finishes writing and verifying the SD card.

Eject the SD card safely.

### 1.6 Boot the Raspberry Pi

Insert the SD card into the Raspberry Pi.

Power on the Raspberry Pi.

Wait 1–2 minutes for the first boot.

Find the Pi IP address from your router or by pinging the hostname:

```bash
ping drone-pi-01.local
```

Example IP:

```text
192.168.123.49
```

### 1.7 Test SSH manually

From your Mac or Linux machine:

```bash
ssh admin@192.168.123.49
```

Use the initial password:

```text
adminadmin
```

If SSH works, the Raspberry Pi is ready for Ansible.

---

## 2. What Ansible does

Ansible connects to the Raspberry Pi over SSH and applies the required system configuration automatically.

### 2.1 Connect to the Pi over SSH

Ansible uses the IP address from `inventory.ini`.

Example:

```ini
drone-pi-01 ansible_host=192.168.123.49
```

### 2.2 Become root using sudo

Many operations require root permissions.

The playbook uses:

```yaml
become: true
```

This means Ansible runs administrative commands using `sudo`.

### 2.3 Install required packages

The playbook installs common system packages:

```text
NetworkManager
ModemManager
SSH server
tcpdump
curl
git
vim/nano
net-tools
iproute2
usb-modeswitch
libqmi-utils
udhcpc
Python 3
pip
venv
```

It also installs MAVLink-related tools:

```text
mavlink-router
MAVProxy
pymavlink
```

These are used for Pixhawk/ArduPilot/MAVLink communication.

### 2.4 Configure SSH

The playbook configures SSH to listen on:

```text
port 22
port 2222
```

It also makes SSH listen on all network interfaces:

```text
0.0.0.0
::
```

This allows SSH through Wi-Fi and through the LTE static IP.

The generated SSH config is written to:

```text
/etc/ssh/sshd_config.d/99-ports.conf
```

Expected result:

```bash
sudo ss -tlnp | grep -E ':22|:2222'
```

Example output:

```text
0.0.0.0:22
0.0.0.0:2222
[::]:22
[::]:2222
```

### 2.5 Configure LTE

The playbook creates a NetworkManager GSM/LTE connection:

```text
Connection name: team-lte
APN: internet
Autoconnect: enabled
```

The LTE modem is managed by:

```text
NetworkManager
ModemManager
```

For the current Raspberry Pi, the LTE modem interface is:

```text
wwp1s0u1u3i2
```

The static LTE IP is:

```text
178.160.252.170
```

### 2.6 Configure LTE policy routing

The Raspberry Pi may have both Wi-Fi and LTE active at the same time.

Wi-Fi is usually used for local management:

```text
192.168.123.49
```

LTE is used for public static IP access:

```text
178.160.252.170
```

Without policy routing, packets received through LTE can be answered through Wi-Fi. That breaks SSH through the public LTE IP.

The playbook fixes this by:

```text
Disabling rp_filter
Creating policy routing table 100
Creating routing rule:
    from 178.160.252.170 use table 100
Creating table 100 default route through LTE gateway
```

For the current Pi:

```text
LTE IP:      178.160.252.170
LTE gateway: 178.160.252.169
LTE network: 178.160.252.168/30
LTE table:   100
```

So replies from:

```text
178.160.252.170
```

go back through:

```text
wwp1s0u1u3i2
```

not through Wi-Fi.

### 2.7 Configure MAVLink router

The playbook writes MAVLink router config to:

```text
/etc/mavlink-router/main.conf
```

It can configure endpoints such as:

```text
QGroundControl on Mac
local swarmkit agent
Pixhawk serial device
```

Example endpoints:

```text
QGC Mac:        192.168.123.46:14550
swarmkit agent: 127.0.0.1:14601
```

The playbook can optionally enable:

```text
mavlink-router.service
```

By default, the service may remain disabled until the Pixhawk/serial setup is ready.

### 2.8 Set hostname

Each Pi gets its own hostname.

Example:

```text
drone-pi-01
```

Other Pis:

```text
drone-pi-02
drone-pi-03
```

---

## 3. Basic requirements on the Raspberry Pi

Before Ansible can configure the Raspberry Pi, the Pi must have a minimal reachable OS.

Required initial state:

```text
Raspberry Pi OS Lite installed
SSH enabled
Wi-Fi or Ethernet connected
Known IP address
User account, for example admin or artyom
User has sudo access
Python 3 installed
Internet access
```

Usually Raspberry Pi OS Lite already has Python 3 installed.

You can check manually:

```bash
python3 --version
```

If Python is missing:

```bash
sudo apt update
sudo apt install -y python3
```

The Pi needs internet access because Ansible installs packages using `apt` and Python packages using `pip`.

Initial internet can be through:

```text
Wi-Fi
Ethernet
LTE
```

For first provisioning, Wi-Fi is usually easiest.

---

## 4. Install Ansible on your Mac

Install Ansible using Homebrew:

```bash
brew install ansible
```

Check:

```bash
ansible --version
```

Install required Ansible collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

---

## 5. Install Ansible on Ubuntu/Linux

On Ubuntu/Debian:

```bash
sudo apt update
sudo apt install -y ansible
```

Check:

```bash
ansible --version
```

Install required Ansible collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

---

## 6. Project files

The project has this structure:

```text
raspi-provisioning/
├── inventory.ini
├── playbook.yml
├── requirements.yml
├── ansible.cfg
├── group_vars/
│   └── all.yml
├── host_vars/
│   ├── drone-pi-01.yml
│   ├── drone-pi-02.yml.placeholder
│   └── drone-pi-03.yml.placeholder
└── roles/
    ├── raspi_base/
    └── mavlink_tools/
```

Common settings for all Raspberry Pis are in:

```text
group_vars/all.yml
```

Per-device settings are in:

```text
host_vars/
```

Example:

```text
host_vars/drone-pi-01.yml
```

---

## 7. Current Raspberry Pi configuration

The current real Raspberry Pi is configured in:

```text
host_vars/drone-pi-01.yml
```

Example:

```yaml
hostname: drone-pi-01

lte_iface: wwp1s0u1u3i2
lte_ip: 178.160.252.170
lte_prefix: 30
lte_gateway: 178.160.252.169
lte_network: 178.160.252.168/30
lte_table: 100
lte_apn: internet

mavlink_router_endpoints:
  - name: qgc_mac
    type: udp
    address: 192.168.123.46
    port: 14550

  - name: swarmkit_agent_local
    type: udp
    address: 127.0.0.1
    port: 14601
```

---

## 8. Test SSH before running Ansible

From your Mac:

```bash
ssh -p 2222 artyom@192.168.123.49
```

For the drone Raspberry Pi setup, use user `admin`:

```bash
ssh admin@192.168.123.49
```

Password:

```text
adminadmin
```

If this works, test Ansible.

---

## 9. Test Ansible connectivity

From inside this directory:

```bash
ansible -i inventory.ini drone-pi-01 -m ping
```

If password login is needed:

```bash
ansible -i inventory.ini drone-pi-01 -m ping --ask-pass
```

If sudo also asks for a password:

```bash
ansible -i inventory.ini drone-pi-01 -m ping --ask-pass --ask-become-pass
```

Expected result:

```text
pong
```

---

## 10. Run provisioning for the current Pi

If SSH key is configured:

```bash
ansible-playbook -i inventory.ini playbook.yml --limit drone-pi-01 --ask-become-pass
```

If SSH key is not configured yet and password login is needed:

```bash
ansible-playbook -i inventory.ini playbook.yml --limit drone-pi-01 --ask-pass --ask-become-pass
```

---

## 11. Verify after provisioning

On the Raspberry Pi:

```bash
hostnamectl
sudo ss -tlnp | grep -E ':22|:2222'
nmcli device status
nmcli connection show team-lte
ip -4 addr show wwp1s0u1u3i2
ip rule
ip route show table 100
ip route get 217.113.25.21 from 178.160.252.170
curl -4 --interface wwp1s0u1u3i2 --connect-timeout 10 http://ifconfig.me
```

Expected public IP result:

```text
178.160.252.170
```

From your Mac, test SSH through LTE static IP:

```bash
ssh -p 2222 artyom@178.160.252.170
```

For the standard drone setup, use user `admin`:

```bash
ssh -p 2222 admin@178.160.252.170
```

Password:

```text
adminadmin
```

---

## 12. Add a new Raspberry Pi

### 12.1 Add the Pi to inventory

Edit:

```text
inventory.ini
```

Example:

```ini
[raspis]
drone-pi-01 ansible_host=192.168.123.49
drone-pi-02 ansible_host=192.168.123.50
```

### 12.2 Create host variables

Copy one of the placeholder files:

```bash
cp host_vars/drone-pi-02.yml.placeholder host_vars/drone-pi-02.yml
```

Edit:

```text
host_vars/drone-pi-02.yml
```

Fill in:

```yaml
hostname: drone-pi-02

lte_iface: wwp1s0u1u3i2
lte_ip: 178.160.252.X
lte_prefix: 30
lte_gateway: 178.160.252.Y
lte_network: 178.160.252.Z/30
lte_table: 100
lte_apn: internet
```

Also set MAVLink ports for this Pi:

```yaml
mavlink_router_endpoints:
  - name: qgc_mac
    type: udp
    address: 192.168.123.46
    port: 14550

  - name: swarmkit_agent_local
    type: udp
    address: 127.0.0.1
    port: 14602
```

### 12.3 Run Ansible for the new Pi

```bash
ansible-playbook -i inventory.ini playbook.yml --limit drone-pi-02 --ask-pass --ask-become-pass
```

---

## 13. MAVLink router service

The playbook installs a template config:

```text
/etc/mavlink-router/main.conf
```

The playbook can enable:

```text
mavlink-router.service
```

By default, the service is disabled in:

```text
group_vars/all.yml
```

Default:

```yaml
mavlink_router_enable_service: false
```

Enable it per host or globally when ready:

```yaml
mavlink_router_enable_service: true
```

Default endpoints are configured in host variables.

Example:

```yaml
mavlink_router_endpoints:
  - name: qgc
    type: udp
    address: 192.168.123.46
    port: 14550

  - name: swarmkit_agent
    type: udp
    address: 127.0.0.1
    port: 14601
```

For real Pixhawk over UART, configure:

```yaml
mavlink_router_uart_device: /dev/ttyAMA0
mavlink_router_uart_baud: 921600
```

For Pixhawk over USB serial, configure:

```yaml
mavlink_router_uart_device: /dev/ttyACM0
mavlink_router_uart_baud: 115200
```

Then enable the service:

```yaml
mavlink_router_enable_service: true
```

---

## 14. Debugging LTE SSH

If ping works but SSH does not, run this on the Pi:

```bash
sudo tcpdump -ni wwp1s0u1u3i2 'tcp port 22 or tcp port 2222 or icmp'
```

Then from Mac:

```bash
ping 178.160.252.170
ssh -v -p 2222 artyom@178.160.252.170
```

If tcpdump shows incoming SYN packets but SSH does not connect, check route decision:

```bash
ip route get <mac_public_ip> from 178.160.252.170
```

It must use LTE:

```text
dev wwp1s0u1u3i2
```

not Wi-Fi:

```text
dev wlan0
```

If it uses Wi-Fi, policy routing is not applied correctly.

Check:

```bash
ip rule
ip route show table 100
```

Expected:

```text
from 178.160.252.170 lookup 100
default via 178.160.252.169 dev wwp1s0u1u3i2
```

---

## 15. Recommended workflow for many Raspberry Pis

For each new Pi:

1. Flash Raspberry Pi OS Lite.
2. Set unique hostname.
3. Set username to `admin`.
4. Set password to `adminadmin`.
5. Enable SSH.
6. Configure Wi-Fi.
7. Boot Pi.
8. Find its Wi-Fi IP.
9. Add it to `inventory.ini`.
10. Create `host_vars/drone-pi-XX.yml`.
11. Run Ansible.

After that, all Pis are configured consistently.

---

## 16. Ansible vs Docker

Ansible should configure the host OS:

```text
SSH
Wi-Fi
LTE modem
NetworkManager
ModemManager
policy routing
systemd services
MAVLink router package
```

Docker should be used later for application services:

```text
swarmkit agent
MAVLink parser
gRPC services
telemetry uploader
mission-control agent
```

Host-level networking should not be hidden inside Docker.

The recommended split is:

```text
Ansible:
  configure the board

Docker:
  run the swarm application
```
