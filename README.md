# Raspberry Pi Swarm Node Provisioning

This Ansible project provisions Raspberry Pi OS Lite drone/swarm nodes over their
initial Wi-Fi SSH connection. Each Pi may have its own hostname and
operator-assigned static LTE address while using the same Telit LE910C4-EU modem
(`1bc7:1201`).

The playbook installs base tools, safely configures SSH on ports `22` and `2222`,
discovers the modem interface dynamically (`wwan0`, `wwp...`, or another
`qmi_wwan` name), activates the `team-lte` NetworkManager profile, configures
persistent source policy routing, and installs MAVProxy/pymavlink/mavlink-router.
It also prepares a native SwarmKit build workspace with Git, Conan 2, CMake,
Ninja, and the C++ compiler toolchain.

NetworkManager is never stopped or restarted by this project during provisioning,
so the Wi-Fi SSH management path remains available. If modem discovery is slow,
only ModemManager may be restarted after a bounded retry period.

## 1. Flash A Raspberry Pi

1. Install [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
2. Select the current **Raspberry Pi OS Lite (64-bit)** based on Debian 13
   (Trixie), and the target SD card. Older Bookworm images ship an older default
   compiler than SwarmKit requires.
3. In OS customization, give each board a unique hostname, such as
   `drone-pi-01`, `drone-pi-02`, or `drone-pi-03`.
4. Set the initial username to `admin` and password to `adminadmin`.
5. Enable SSH and configure the Wi-Fi SSID, Wi-Fi password, and country.
6. Write the image, insert the card, boot the Pi, and wait for initial startup.

The shared bootstrap password is suitable for initial lab setup only. Move to
SSH keys and protected credentials before field deployment.

Find the Pi's Wi-Fi address in the router UI or try:

```bash
ping drone-pi-01.local
ssh admin@drone-pi-01.local
```

For the current first node the Wi-Fi address is `192.168.123.49`:

```bash
ssh admin@192.168.123.49
```

Use `adminadmin` at the initial password prompt.

## 2. Configure Per-Pi Values

`inventory.ini` contains Wi-Fi SSH addresses used during provisioning:

```ini
[raspis]
drone-pi-01 ansible_host=192.168.123.49

[raspis:vars]
ansible_user=admin
ansible_port=22
```

Keep the LTE values unique per node in `host_vars/<hostname>.yml`. The current
`host_vars/drone-pi-01.yml` is:

```yaml
---
hostname: drone-pi-01
lte_ip: 178.160.252.170
lte_prefix: 30
lte_gateway: 178.160.252.169
lte_network: 178.160.252.168/30
lte_table: 100
lte_apn: internet
```

Copy a placeholder for additional boards and fill in the public IP allocation
provided by the operator:

```bash
cp host_vars/drone-pi-02.yml.placeholder host_vars/drone-pi-02.yml
cp host_vars/drone-pi-03.yml.placeholder host_vars/drone-pi-03.yml
```

Do not set an LTE interface normally. The role reads the ModemManager net port
and falls back to the `qmi_wwan` sysfs device. For an unusual device requiring a
manual recovery override, add this only in that Pi's host vars:

```yaml
lte_iface_override: wwan0
```

SwarmKit source cloning is optional because the repository may need private
credentials. By default Ansible creates `/home/admin/workspace`; after
provisioning you can clone into `/home/admin/workspace/swarmkit`. If the Pi
already has Git access to the repository, configure this in protected variables:

```yaml
swarmkit_repo_url: git@github.com:YOUR_ACCOUNT/swarmkit.git
swarmkit_repo_version: main
```

Override `swarmkit_build_user`, `swarmkit_build_group`, and
`swarmkit_build_home` together only if a node uses a build account other than
the standard `admin` user.

## 3. Install Ansible On The Controller

On macOS:

```bash
brew install ansible
```

On Debian or Ubuntu:

```bash
sudo apt update
sudo apt install -y ansible sshpass
```

From this project directory install the declared Ansible collection:

```bash
ansible-galaxy collection install -r requirements.yml
```

## 4. Provision Over Wi-Fi

Confirm that Ansible can log in to the freshly flashed board. At password and
sudo/become prompts, enter `adminadmin` unless credentials have already been
changed:

```bash
ansible raspis -m ping --ask-pass --ask-become-pass
```

Provision the node:

```bash
ansible-playbook playbook.yml --ask-pass --ask-become-pass
```

The playbook performs the following checks rather than silently continuing:

- ModemManager must discover a Telit modem after retries and, if necessary, a
  safe ModemManager-only restart.
- `team-lte` must become active and receive the expected operator-assigned IP.
- Traffic sourced from the LTE IP must route through policy table `100` and the
  detected LTE interface.
- `mavproxy.py --version` and `mavlink-routerd` must be available after install.
- SwarmKit prerequisites must include CMake `>= 3.28,<4`, Conan 2, Ninja, Git,
  and G++ `>= 13`, and `/home/admin/workspace` must be prepared.

This project deliberately installs the Python command-line applications
`MAVProxy`, `pymavlink`, `future`, Conan, and modern CMake globally with
Debian's `--break-system-packages` opt-in. A dedicated freshly flashed node
does not need per-tool virtual environments; Ansible owns these global tool
installations.

If modem or LTE activation fails, the failed run prints kernel, USB,
NetworkManager, and ModemManager evidence. More can be collected at any time:

```bash
ansible-playbook diagnostics.yml --ask-pass --ask-become-pass
```

## 5. Verify The Node

Run the verification playbook after provisioning or after reboot:

```bash
ansible-playbook verify.yml --ask-pass --ask-become-pass
```

It checks hostname, SSH listeners, DNS/internet access, Telit USB and
ModemManager detection, active LTE address, LTE public egress IP, routing rules
and table, both MAVLink tools, and the SwarmKit build toolchain/workspace.

For `drone-pi-01`, successful output ends with values similar to:

```text
Hostname: drone-pi-01
LTE interface/address: wwan0 / 178.160.252.170
LTE public IP check: 178.160.252.170
Route decision: 1.1.1.1 via 178.160.252.169 dev wwan0 table 100 from 178.160.252.170
MAVProxy: /usr/local/bin/mavproxy.py
mavlink-routerd: /usr/local/bin/mavlink-routerd
SwarmKit workspace: /home/admin/workspace
Build tools: cmake version >=3.28,<4, Conan version 2.x, G++ 14.x
```

The interface may instead be named `wwp...`; that is expected and is handled
automatically.

## 6. Build SwarmKit On The Pi

The `swarmkit_build` role installs Git and everyday node tools such as `vim`,
along with `build-essential`, GCC/G++, Ninja, `pkg-config`, common dependency
build helpers, Conan 2, and CMake `>= 3.28,<4`. It initializes the Conan
default profile for `admin` and creates `/home/admin/workspace/README.build.txt`.

SwarmKit's `linux-debug` and `linux-release` presets are suitable for a native
Raspberry Pi ARM64 build. The current `scripts/ci_package_linux_x86_64.sh`
script is specifically for x86_64 packages; do not use it to publish Pi
artifacts until it is made architecture-aware.

If `swarmkit_repo_url` was not configured, log in to the Pi and clone the
source:

```bash
ssh admin@192.168.123.49
cd ~/workspace
git clone <your-swarmkit-git-url> swarmkit
cd swarmkit
```

Build the project on Raspberry Pi OS:

```bash
# Debug
conan install . -of build/conan -s build_type=Debug -s compiler.cppstd=23 --build=missing
cmake --preset linux-debug
cmake --build --preset linux-debug

# Release and tests
conan install . -of build/conan -s build_type=Release -s compiler.cppstd=23 --build=missing
cmake --preset linux-release
cmake --build --preset linux-release
ctest --preset linux-release --output-on-failure
```

The first Conan build on ARM64 may compile gRPC, Protobuf, and other
dependencies locally, so allow time and free disk space. If a Raspberry Pi OS
image provides G++ older than version 13, provisioning fails with that
diagnosis; use a current 64-bit image with a C++23-capable compiler.

## 7. Test SSH Through LTE

After verification succeeds and the operator permits inbound public-IP traffic,
test from an external network:

```bash
ssh admin@178.160.252.170
ssh -p 2222 admin@178.160.252.170
```

Replies to inbound LTE SSH connections stay on LTE because the playbook:

- disables reverse path filtering for the relevant interfaces;
- writes a NetworkManager source rule for the LTE IP in routing table `100`;
- installs `lte-policy-routing.service` as a boot-time repair path for modem
  activation timing.

## 8. MAVLink Router Service

MAVProxy, pymavlink, `future`, and `mavlink-routerd` are installed system-wide.
No MAVLink virtual environment is created. If Raspberry Pi OS does not publish
`mavlink-router`, the role builds it from the official source repository with
Meson/Ninja.

The role writes `/etc/mavlink-router/main.conf`, but leaves the router service
disabled by default until a flight controller and endpoint configuration are
ready. Enable it for a node in host vars:

```yaml
mavlink_router_enable_service: true
```

## 9. Host Networking Versus Applications

Ansible owns host-level provisioning: packages, SSH, modem detection,
NetworkManager, policy routing, and MAVLink router integration. Docker should
later run application services such as `swarmkit`; it should not be used to
configure the Pi's LTE modem or host policy routing.
