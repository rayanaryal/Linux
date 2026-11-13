# Installing Ubuntu Server on an Old WatchGuard Firewall

My notes from turning an old **WatchGuard Firebox XTM 5 Series** firewall into an **Ubuntu Server**, controlled over the **serial console** and **SSH**.

> **Placeholders:** replace these with your own values.
> - `<firewall-ip>`: the firewall's IP address on your network (e.g. `192.168.x.x`)
> - `<username>`: the user you created during the Ubuntu install
> - `<hostname>`: the server name you chose during the install

---

## Table of Contents

1. [The Hardware](#1-the-hardware)
2. [Connecting with the Serial Console](#2-connecting-with-the-serial-console)
3. [Reading the BIOS](#3-reading-the-bios)
4. [The Boot Problem (Freezing at 0078)](#4-the-boot-problem-freezing-at-0078)
5. [Fitting the SSD](#5-fitting-the-ssd)
6. [Installing Ubuntu onto the SSD with QEMU](#6-installing-ubuntu-onto-the-ssd-with-qemu)
7. [Turning On the Serial Console in GRUB](#7-turning-on-the-serial-console-in-grub)
8. [First Boot on the Firewall](#8-first-boot-on-the-firewall)
9. [Setting Up the Network (Netplan)](#9-setting-up-the-network-netplan)
10. [Updating the System](#10-updating-the-system)
11. [SSH: Logging In Over the Network](#11-ssh-logging-in-over-the-network)
12. [SSH Keys](#12-ssh-keys)
13. [Turning Off Password Logins](#13-turning-off-password-logins)
14. [Giving Someone Else Access](#14-giving-someone-else-access)
15. [Mistakes I Made (and Lessons)](#15-mistakes-i-made-and-lessons)
16. [Command Cheat Sheet](#16-command-cheat-sheet)
17. [Next Steps](#17-next-steps)

---

## 1. The Hardware

| Item | Details |
|---|---|
| **Device** | WatchGuard Firebox **XTM 5 Series** (1U rack, LCD on front) |
| **Hardware model** (sticker) | **NC2AE8** |
| **Motherboard** (BIOS screen) | **MB-7580** |
| **CPU** | Intel **Celeron 440 @ 2.00 GHz** (64-bit x86, so normal Ubuntu works) |
| **RAM** | **1 GB DDR2 800** (one stick; a second slot is free) |
| **Storage slot** | **CF (CompactFlash)** slot, empty, plus a **SATA** port |
| **Network ports** | 7 x Gigabit Ethernet |
| **Console port** | **RJ45**, the leftmost port next to the USB ports (it has **no LEDs**) |
| **BIOS** | AMI BIOS, locked by WatchGuard ("WG BIOS V1.2" on the LCD) |

**Other stickers explained:**
- **Anatel**: Brazilian approval mark (not the brand)
- **VCCI-A**: Japanese approval mark; Class A = business/industrial equipment

**Tip:** you can tell the console port apart because the network ports have **two little LEDs** each, and the console port has **none**.

---

## 2. Connecting with the Serial Console

The firewall has **no screen output** (no VGA/HDMI). You see its text through a **console cable**.

### The cable

- **USB-to-RJ45 console cable** (mine has an **FTDI** chip, the most reliable kind on Linux)
- **USB end → laptop**
- **RJ45 end → the firewall's Console port** (leftmost, no LEDs)

> ⚠️ If the cable is in a **network** port instead, you get endless garbage characters.

### Check the laptop sees the cable

```bash
lsusb                      # look for FTDI / Prolific / CH340 / Silicon Labs
ls /dev/ttyUSB*            # should show /dev/ttyUSB0
sudo dmesg | grep ttyUSB   # shows which chip, e.g. "FTDI USB Serial Device converter now attached to ttyUSB0"
```

- On Linux, devices show up as files in `/dev/`.
- `ttyS0`–`ttyS31` are old built-in serial port names; a laptop doesn't have real ones, so ignore them.
- If a line mentions **brltty** (a braille program that can steal USB serial cables): `sudo apt remove brltty`

### Baud rate (speed)

- **115200** for this WatchGuard (8 data bits, no parity, 1 stop bit = **8N1**).
- Cisco devices usually use **9600**.
- Both sides must use the **same speed**, or you see garbage.
- If the garbage looks **the same at every speed**, it's **not** a speed problem. Check the **port** and the **cable**.

### picocom (my preferred serial program)

```bash
sudo apt install picocom
clear
sudo picocom -b 115200 /dev/ttyUSB0
```

| Keys | What it does |
|---|---|
| **Ctrl+A, Ctrl+X** | **Exit** picocom |
| Ctrl+A, Ctrl+C | Local echo on/off |
| Ctrl+A, Ctrl+U / Ctrl+D | Speed up / down |
| Ctrl+A, Ctrl+H | Help |

- `Terminal ready` means you're connected. Your keys now go **to the firewall**, not the laptop.
- Your normal laptop prompt (e.g. `you@laptop:~$`) means you're on the **laptop**. `Terminal ready` means you're **talking to the firewall**.

### screen (alternative)

```bash
sudo screen /dev/ttyUSB0 115200     # leaving out 115200 makes it use 9600!
```
Exit: **Ctrl+A**, then **K**, then **Y**.

`[screen is terminating]` straight away means screen couldn't open the port.

### Checking whether something else is using the cable

```bash
sudo fuser -v /dev/ttyUSB0     # no output = nothing is using it (exit code 1 is normal here)
```

---

## 3. Reading the BIOS

What the firewall printed at power-on:

```
AMIBIOS(C)2006 American Megatrends, Inc.
MB-7580 Ver.WC0 02/03/2010
CPU : Intel(R) Celeron(R) CPU 440 @ 2.00GHz
Press DEL to run Setup (F4 on Remote Keyboard)
Press n if you want to boot from the network
Press F11 for BBS POPUP (F3 on Remote Keyboard)
The MCH is operating with DDR2 800
990MB OK
```

| Line | Meaning |
|---|---|
| `Celeron 440` | 64-bit Intel CPU, so Ubuntu can run |
| `DDR2 800` | RAM type |
| `990MB OK` | About 1 GB of RAM (a little is reserved by the board) |
| `F4 on Remote Keyboard` | Over the console cable, **F4** = BIOS setup |
| `F3 on Remote Keyboard` | Over the console cable, **F3** = boot menu |
| `Press n` | Network boot (PXE) |
| `3rd Master : TS32GSSD25S-M` | The SSD is detected |
| `0078` (bottom-right corner) | A **POST progress code**; if it stays on one code, that's where it's stuck |

### Function keys on my Dynabook

- **F3 on its own** sends a real F3 (`^[OR`).
- **Fn+F3** is the laptop's **sleep/lock** shortcut, so don't use Fn.
- Test what a key sends: run `cat -v`, press the key, then **Ctrl+C** to quit.

### luit: fixing garbled BIOS menus

Old BIOS menus draw boxes using the old DOS character set **CP437**. Ubuntu uses **UTF-8**, so the boxes show as junk. **luit** translates between them:

```bash
sudo apt install luit
sudo luit -encoding CP437 -- picocom -b 115200 /dev/ttyUSB0
```

- `--` means "my luit settings end here; run this program next".
- Only needed for the **BIOS**. For Linux itself, use plain picocom.

---

## 4. The Boot Problem (Freezing at 0078)

**Symptom:** with no bootable drive, the BIOS always stopped at POST code **0078** and never showed the boot menu, so F3, F4 and network boot (`n`) couldn't help.

| Tried | Result |
|---|---|
| Ubuntu USB stick in the firewall | Froze at 0078 |
| No USB stick | Froze at 0078 |
| Network boot (`n`) | Accepted ("Network selected as first boot device"), but still froze at 0078 |
| Empty SSD connected | Detected, then froze at 0078 |
| **SSD with Ubuntu installed on it** | ✅ **Booted normally** |

**Lesson:** I thought it needed a CF card, but once the SSD had a working operating system on it, the BIOS started it fine. **No CF card was needed.**

---

## 5. Fitting the SSD

**SSD:** Transcend 32 GB 2.5" SATA SSD (TS32GSSD25S-M)

An SSD needs **two cables**:

| Cable | Purpose | Look |
|---|---|---|
| **SATA data** | Data to and from the motherboard | Thin, flat (red), small L-shaped plug |
| **SATA power** | Electricity from the power supply | Wider L-shaped plug, black, with yellow/red/black wires |

- On the SSD: the **short** slot is data, the **long** slot is power. They only fit one way.
- ⚠️ My first attempt didn't work because **the power plug wasn't connected**.
- ⚠️ Always **switch off and unplug** the firewall before touching anything inside.

---

## 6. Installing Ubuntu onto the SSD with QEMU

**Why:** the locked BIOS wouldn't boot the installer from USB, so I installed Ubuntu onto the SSD **from my laptop**, then moved the SSD into the firewall.

**QEMU** creates a **pretend computer** in a window. It acts like an **old Legacy BIOS PC**, which is exactly what the firewall needs (not UEFI).

### What you need

- **USB-to-SATA adapter**, to plug the SSD into the laptop
- **Ubuntu Server** installer USB (mine: *Ubuntu-Server 24.04.5 LTS amd64*)

> Ubuntu **Server**, not Desktop: Desktop needs about 4 GB RAM and a screen. Server is text-only and runs fine on 1 GB.

### Step 1: Find the drives

```bash
lsblk -o NAME,LABEL,SIZE
```

| Name | Size | What it was |
|---|---|---|
| `sda` | 29.8G | The SSD (32 GB shows as 29.8G because of how Linux counts) |
| `sdb` | 57.8G | The Ubuntu Server USB stick |
| `nvme0n1` | 238.5G | ⚠️ **My laptop's own drive. Never touch it!** |

### Step 2: Install QEMU

```bash
sudo apt install qemu-system-x86
```

My network has **no IPv6 internet**, and apt failed with `Network is unreachable` on IPv6 addresses. Fix:
```bash
sudo apt -o Acquire::ForceIPv4=true update
sudo apt -o Acquire::ForceIPv4=true install qemu-system-x86
```

### Step 3: Start the installer in QEMU

```bash
sudo qemu-system-x86_64 -enable-kvm -m 2048 -drive file=/dev/sda,format=raw -cdrom /dev/sdb -boot d
```

| Part | Meaning |
|---|---|
| `-enable-kvm` | Use the CPU's hardware boost (fast) |
| `-m 2048` | 2 GB RAM for the pretend computer |
| `-drive file=/dev/sda,format=raw` | Its hard drive = the SSD |
| `-cdrom /dev/sdb` | Its CD drive = the Ubuntu USB stick |
| `-boot d` | Start from the CD drive |

- Release the mouse from the QEMU window: **Ctrl+Alt+G**
- The `CPUID ... svm` warning is harmless.

### Step 4: Installer choices

| Screen | Choice |
|---|---|
| Type of install | **Ubuntu Server** (not minimized) |
| Storage | **Use an entire disk** → the ~30 GB SSD |
| Profile | A server name (hostname) and username of your choice. Write the password down |
| SSH | ✅ **Install OpenSSH server** |
| Snaps | None |

**Good sign in the log:** `Installing packages on target system: ['grub-pc']`. **grub-pc** is the boot loader for **Legacy BIOS**.

At "Please remove the installation medium", just **close the QEMU window**. The `[FAILED] Failed unmounting cdrom` message is harmless.

### Step 5: Check the result

```bash
lsblk
```

| Partition | Size | Purpose |
|---|---|---|
| `sda1` | 1M | BIOS boot area (proves Legacy BIOS install ✅) |
| `sda2` | 2G | `/boot` |
| `sda3` | 27.8G | Main system |

### Step 6: Boot the SSD in QEMU (with 1 GB RAM, like the firewall)

```bash
sudo qemu-system-x86_64 -enable-kvm -m 1024 -drive file=/dev/sda,format=raw
```

---

## 7. Turning On the Serial Console in GRUB

This makes Ubuntu send its text out of the **console port**, so it shows in picocom.

```bash
sudo nano /etc/default/grub
```

Set these lines:
```
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=5
GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200n8"
GRUB_TERMINAL="console serial"
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
```

| Line | Meaning |
|---|---|
| `GRUB_TIMEOUT_STYLE=menu` / `GRUB_TIMEOUT=5` | Show the boot menu for 5 seconds |
| `console=ttyS0,115200n8` | Linux messages go to serial port 0 at 115200 |
| `GRUB_TERMINAL="console serial"` | Show the GRUB menu on the serial port too |
| `GRUB_SERIAL_COMMAND=...` | Serial settings for GRUB (8N1 at 115200) |

nano: save **Ctrl+O**, **Enter**; exit **Ctrl+X**.

Apply:
```bash
sudo update-grub
```

### Fixing a typo with sed

I typed `--party=no` instead of `--parity=no`. Fixed with:
```bash
sudo sed -i 's/--party=no/--parity=no/' /etc/default/grub
grep SERIAL /etc/default/grub      # show only the lines containing SERIAL
sudo update-grub
```
- `sed -i 's/old/new/' file` = find and replace inside a file.
- `grep WORD file` = show only lines containing WORD.

Shut down QEMU when done:
```bash
sudo shutdown now
```

---

## 8. First Boot on the Firewall

1. Move the SSD into the firewall (data + power cables). CF slot left empty.
2. On the laptop (plain picocom, no luit):
   ```bash
   clear
   sudo picocom -b 115200 /dev/ttyUSB0
   ```
3. Power on and watch: BIOS → GRUB → Linux start-up lines → `<hostname> login:`

```
Ubuntu 24.04.5 LTS <hostname> ttyS0
<hostname> login: <username>
```

- `ttyS0` = logged in through the console cable.
- Linux usernames are **case-sensitive**: `myuser` ≠ `Myuser`.

---

## 9. Setting Up the Network (Netplan)

### See the network ports

```bash
ip a
```

| Name | What it is |
|---|---|
| `lo` | Loopback (the computer talking to itself, 127.0.0.1) |
| `enp1s0`, `ens32`–`ens37` | The 7 network ports |
| `link/ether xx:xx:xx:...` | MAC address of each port (the first half is the maker's code) |
| `state DOWN` / `NO-CARRIER` | Not set up / no cable plugged in |

### The problem

The installer set up the network for QEMU's port **`ens3`**, which doesn't exist on the firewall.

### The fix: one rule for every port

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    all-ports:
      match:
        name: "en*"
      dhcp4: true
      optional: true
```

| Line | Meaning |
|---|---|
| `match: name: "en*"` | Apply to every port whose name starts with `en` |
| `dhcp4: true` | Get an IPv4 address automatically from the router |
| `optional: true` | Don't make start-up wait if a cable is unplugged |

> YAML uses **spaces (never Tab)** to show what belongs to what. 2 spaces per level.

```bash
sudo chmod 600 /etc/netplan/50-cloud-init.yaml   # only root can read it
sudo netplan apply
ip a
```

**Result:** the port with the cable → `state UP`, `inet 192.168.x.x/24` (an address from my router by DHCP). I call this `<firewall-ip>` in the rest of these notes.

### Testing

```bash
ping -c 3 8.8.8.8          # IPv4 internet test
```
- `ping -6 -c 3 8.8.8.8` gives *"Address family for hostname not supported"*: `-6` means IPv6, but 8.8.8.8 is an IPv4 address.

### IPv4 vs IPv6 on my network

| IPv6 address starts with | Type | Reaches the internet? |
|---|---|---|
| `fe80::` | **Link-local** (every device makes one itself) | ❌ No |
| `2...` or `3...` | **Global unicast** (from the router/ISP) | ✅ Yes |

My devices only have `fe80::` addresses, so **no IPv6 internet**. That's why apt on my laptop failed on IPv6 and needed IPv4 forced.

To make apt always use IPv4 (only if needed):
```bash
echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4
```
> On the firewall, `apt update` worked fine without it. Good habit: **don't add a fix until you've seen the problem.**

---

## 10. Updating the System

| Command | What it does | Like... |
|---|---|---|
| `sudo apt update` | Downloads the **list** of what's new. Changes nothing | Reading the menu |
| `sudo apt upgrade -y` | **Installs** the newer versions | Ordering the food |

```bash
sudo apt update
apt list --upgradable      # see what will be updated
sudo apt upgrade -y        # -y = answer yes automatically
sudo reboot
```

- `&&` runs the second command only if the first one worked: `sudo apt update && sudo apt upgrade -y`
- `noble` = code name for Ubuntu 24.04. `amd64` = 64-bit Intel/AMD.
- **needrestart** (after upgrading) lists services that were restarted or still need it. A reboot clears everything.
- "New release '26.04.1 LTS' available": ignored on purpose. 24.04 is supported for years. `apt upgrade` does **not** change the Ubuntu version.

**Before:** 29 updates. **After:** `0 updates can be applied immediately` ✅

---

## 11. SSH: Logging In Over the Network

**SSH = Secure Shell.** It lets you control another computer's command line over the network, **encrypted**.

```
Laptop (ssh client)  ══ encrypted ══  Firewall (OpenSSH server, port 22)
```

| SSH's job | What it means |
|---|---|
| **Encryption** | Everything, including passwords, is scrambled on the network |
| **Server identity** | The server shows a **fingerprint** so you know it's the real one |
| **Your identity** | You log in with a password or a key |

- SSH replaced **Telnet**, which sent everything (passwords too) in plain text.
- In **CCNA**: disable Telnet and enable SSH on Cisco devices.

### Logging in

```bash
ssh <username>@<firewall-ip>
```

First time: *"The authenticity of host ... can't be established"*. Check the **fingerprint**, type `yes`, and it's saved to `~/.ssh/known_hosts`. The fingerprint matched the one Ubuntu printed on first boot ✅.

### Error messages

| Error | Usually means |
|---|---|
| **Network is unreachable** | Your own computer has **no route** (Wi-Fi off, wrong network, VPN) |
| **Connection timed out** | A route exists, but the other device isn't answering |
| **Connection refused** | The device answered, but nothing is listening on that port |
| **Permission denied (publickey)** | You're not allowed in (no valid key, and passwords are off) |

> My first SSH attempt failed with "Network is unreachable" because **my laptop's Wi-Fi was off**.

---

## 12. SSH Keys

| | What it is | Where it lives | Share it? |
|---|---|---|---|
| **Private key** (`id_ed25519`) | Your secret key | Only on your own device | ❌ **Never** |
| **Public key** (`id_ed25519.pub`) | The padlock | Copied to servers | ✅ Safe to share |

The server keeps a **guest list** of allowed public keys: `~/.ssh/authorized_keys`.

### Steps (on the device that wants access)

```bash
ls ~/.ssh/                                  # already have id_ed25519.pub? then skip ssh-keygen
ssh-keygen -t ed25519 -C "a-label"          # make a key pair (Enter for defaults)
ssh-copy-id <username>@<firewall-ip>           # copy the PUBLIC key to the server's guest list
ssh <username>@<firewall-ip>                   # test: no firewall password needed
```

| Command | What it does |
|---|---|
| `ssh` | Just **logs in**. Saves nothing |
| `ssh-copy-id` | **Registers** your public key on the server (needs one password login) |

### A public key line has 3 parts (separated by single spaces)

```
ssh-ed25519   <long-key-text>   <label>
   type       the key (no spaces)   comment/label
```
The long key text uses letters that look alike (`l`, `I`, `1`, `O`, `0`), so one wrong character breaks it.

### Checking keys with fingerprints

```bash
ssh-keygen -lf ~/.ssh/id_ed25519.pub        # fingerprint of my key
ssh-keygen -lf ~/.ssh/authorized_keys       # fingerprints of every key on the guest list
cat -n ~/.ssh/authorized_keys               # show the guest list with line numbers
```
If a key **doesn't appear** in `ssh-keygen -lf`, that line is **damaged**, and SSH ignores it too.

### Removing one line from the guest list

```bash
cp ~/.ssh/authorized_keys ~/.ssh/authorized_keys.bak   # backup first!
sed -i '2d' ~/.ssh/authorized_keys                     # delete line 2
cat -n ~/.ssh/authorized_keys                          # check
```

### Adding a key by hand (only if needed)

```bash
echo "<the whole public key line>" >> ~/.ssh/authorized_keys
```
- `>>` **adds** to the end of a file. `>` **replaces** the whole file (dangerous here!).

> ⚠️ **Never retype keys.** My Mac's key broke because a hand-typed copy had a capital **I** where it needed a lowercase **l**. Use `ssh-copy-id` or copy-paste.

---

## 13. Turning Off Password Logins

**Goal:** only devices with a registered key can log in over SSH. This blocks **brute-force** password guessing.

```bash
sudo nano /etc/ssh/sshd_config.d/01-no-passwords.conf
```
```
PasswordAuthentication no
KbdInteractiveAuthentication no
```

| Part | Meaning |
|---|---|
| `sshd` | The SSH **server** (the `d` = daemon, a background service) |
| `sshd_config.d/` | Folder for extra server settings |
| `01-` | Loads first, so it wins over other files |

```bash
sudo sshd -t                     # check for mistakes (no output = OK)
sudo systemctl restart ssh       # apply
```

### Testing (from a device with a key)

```bash
ssh <username>@<firewall-ip>                              # ✅ logs in with the key
ssh -o PubkeyAuthentication=no <username>@<firewall-ip>   # ✅ Permission denied (publickey)
```

### Adding a new device later (the passwords on → copy key → passwords off cycle)

On the firewall (each command on its own line):
```bash
sudo sed -i 's/ no$/ yes/' /etc/ssh/sshd_config.d/01-no-passwords.conf
sudo systemctl restart ssh
```
On the new device:
```bash
ssh-copy-id <username>@<firewall-ip>
```
Back on the firewall:
```bash
sudo sed -i 's/ yes$/ no/' /etc/ssh/sshd_config.d/01-no-passwords.conf
cat /etc/ssh/sshd_config.d/01-no-passwords.conf
sudo systemctl restart ssh
```
(`$` in sed = end of the line.)

### What still uses a password

- The **console cable** (emergency way in)
- **`sudo`**

### Devices with access now

- 💻 My Linux laptop (its own key)
- 🍎 My Mac (its own key)

> Keep a **backup** of private keys somewhere safe. If you lose them, the console cable is the way back in.

---

## 14. Giving Someone Else Access

**Never share your own key or password.** Give each person **their own account and key**.

| Shared login ❌ | Own account ✅ |
|---|---|
| Can't tell who did what | Linux logs who logged in |
| Removing access means changing **your** key | Just delete **their** account/key |
| They get all your powers | You choose their powers |

This is the **principle of least privilege**: only give the access someone needs.

```bash
sudo adduser boss
sudo mkdir -p /home/boss/.ssh
sudo nano /home/boss/.ssh/authorized_keys     # paste THEIR public key
sudo chown -R boss:boss /home/boss/.ssh       # make them the owner
sudo chmod 700 /home/boss/.ssh                # only they can open the folder
sudo chmod 600 /home/boss/.ssh/authorized_keys
```

- **View only:** do nothing else (they can read most of `/etc` but not change it).
- **Admin:** `sudo usermod -aG sudo boss`
- **Remove later:** `sudo deluser --remove-home boss`

Other options: show it in person over SSH, or keep config files in a **Git** repo so changes are tracked.

> Access from outside the home network needs a **VPN**. Never open SSH directly to the internet.

---

## 15. Mistakes I Made (and Lessons)

| Mistake | Lesson |
|---|---|
| Console cable plugged in backwards / into a network port | USB end → laptop, RJ45 → **Console** port (no LEDs) |
| `screen` without `115200` | Leaving out the speed defaults to 9600 |
| Pressed **Fn+F3** | On my laptop that's sleep. Plain **F3** sends F3 |
| `-- picocom` typed as `--picocom` | **Spaces separate words** in a command |
| `lsblk -o NAME, LABEL, SIZE` | Lists after an option are one word: `NAME,LABEL,SIZE` |
| `file=/dev/sda, format=raw` | Same: no space after the comma |
| `sudo nano/etc/...` | Missing space: needs `nano /etc/...` |
| `ssh -copy-id` | It's one program: `ssh-copy-id` |
| `--party=no` | Typos in config files break things. Check with `grep` |
| Three commands typed on one line | **One command per line** (or join with `&&`) |
| Typed my username with a capital letter | Usernames are **case-sensitive** |
| Logged in with `ssh` instead of `ssh-copy-id` | Logging in ≠ registering a key |
| Hand-typed SSH key | `l`, `I` and `1` look alike. Never retype keys |
| Wi-Fi off | "Network is unreachable" = check your own connection first |
| SSD power plug not connected | Drives need **data + power** |

**Also:** an add-on in my terminal prints orange "roast" messages after errors. They're **not from Linux** and are often wrong. Trust the **white** output.

---

## 16. Command Cheat Sheet

### Serial console
```bash
ls /dev/ttyUSB*                                   # find the console cable
sudo picocom -b 115200 /dev/ttyUSB0               # connect (exit: Ctrl+A, Ctrl+X)
sudo luit -encoding CP437 -- picocom -b 115200 /dev/ttyUSB0   # for BIOS menus
```

### Drives
```bash
lsblk                                             # list drives
lsblk -o NAME,LABEL,SIZE                          # with labels
```

### Network
```bash
ip a                                              # ports and IP addresses
ip route                                          # routing table
ping -c 3 8.8.8.8                                 # test internet (IPv4)
sudo netplan apply                                # apply network settings
```

### Updates
```bash
sudo apt update                                   # refresh the list
apt list --upgradable                             # see what's new
sudo apt upgrade -y                               # install updates
sudo reboot                                       # restart
sudo shutdown now                                 # power off
```

### SSH
```bash
ssh <username>@<firewall-ip>                         # log in
ssh-keygen -t ed25519 -C "label"                  # make a key pair
ssh-copy-id <username>@<firewall-ip>                 # register your key on the server
ssh-keygen -lf ~/.ssh/authorized_keys             # fingerprints on the guest list
sudo sshd -t                                      # check SSH server config
sudo systemctl restart ssh                        # restart SSH server
```

### Files and text
```bash
cat file                                          # show a file
cat -n file                                       # show with line numbers
grep WORD file                                    # show lines containing WORD
sed -i 's/old/new/' file                          # find and replace
sed -i '2d' file                                  # delete line 2
cp file file.bak                                  # make a backup
nano file                                         # edit (save Ctrl+O, exit Ctrl+X)
```

---

## 17. Next Steps

- [ ] Give the firewall a **fixed IP** (DHCP reservation in the router, or a static IP in netplan)
- [ ] Set up **`ufw`** (allow SSH, block everything else)
- [ ] Decide what the box becomes:
  - **Router/firewall:** WAN + LAN ports, routing, NAT, DHCP server, firewall rules (CCNA practice)
  - **Linux home lab:** users, permissions, Git
  - **Network services:** Pi-hole, WireGuard VPN, monitoring
- [ ] Optional: add a second **1 GB DDR2 800** RAM stick (2 GB total)
- [ ] Back up the **private SSH keys**
