# Virtual Home Lab — Ubuntu Server, SSH, and Improvisations After Improvisations

> How a simple RAM and performance shortage turned a simple two-VM exercise into an important lesson on host-only networking, patience, and knowing when and how to change the plan.


## TL;DR

The original plan was: two VMs, Ubuntu Server + Windows 11, talking to each other over an internal network, SSH'd and hardened. The plan met my 8GB of RAM. The plan lost. What actually shipped: Ubuntu Server VM talking to my Windows 11 **host** instead of a second guest, over a Host-Only adapter, sby a static up, SSH-hardened, and Fail2ban'd. Same learning objectives, half the RAM bill, half the insanity too.

>If you're here from my resume/portfolio: this repo documents the adapted, working setup, the config files behind it, and the (many, dumb) problems I ran into along the way — because that's the actual job. Meanwhile i have documented the entire methodology in even more detail in my Medium Blog with a similar title:
>medium blog URL: https://medium.com/@antarjit05/provisioning-my-home-lab-ubuntu-server-ssh-security-and-improvisations-after-improvisations-1df076b41267?sharedUserId=antarjit05


## Architecture
Full illustrated network diagram: [`docs/network-diagram.png`](docs/network-diagram.md)


## Why the plan changed

Original roadmap: spin up two VMs: Ubuntu Server and Windows 11 — connect them over an internal network, assign static IPs, get SSH working, harden, snapshot, document. Classic networking fundamentals, nothing fancy. It went the intended way for quite some time.

Then I actually tried booting the Windows 11 VM.

My machine has 8GB of RAM total. Native Windows 11 was already eating a chunk of that before any VM entered the picture, leaving me with something like 5.7GB usable. Windows 11 is, and I cannot stress this enough, a freakishly RAM-hungry OS. Task Manager confirmed almost 1.9GB gone to a single VM the moment it powered on, and usage sat at full capacity, meanwhile i had allocated like 4gb to the VM. So it reached 100% RAM usage before it could utilize 4gb anyway. It wasn't a config problem, it wasn't a driver problem. It was just math, and budget.

(Perhaps 2026 was maybe not the ideal year to pick up virtualization and cybersecurity as hobbies, RAM/GPU market being what it is. Micron, if you're reading this, my consumer wallet cries out to you man...)

So I made the call every homelabber eventually makes: adapt the lab instead of abandoning it. Since my host is already Windows 11, I dropped the second VM and had the Ubuntu Server VM talk to the **host PC** instead — same protocol (SSH), same learning objectives (networking fundamentals, linux terminal traversal, static IPs, hardening, snapshots), half the RAM footprint. The Windows 11 VM remains only as a historical artifact. It was configured, it was attempted, it never got to boot as part of a functional network.


## What I actually did (also kinda TL;DR'd)

### 1. VirtualBox + Ubuntu Server setup
- Oracle VirtualBox 7.2.12 on the host
- Ubuntu Server 26.04 LTS installed as a guest VM (2GB base RAM, 1 vCPU, 25GB disk to be allocated at start)
- Learned the obvious-in-hindsight fact that `.iso` files are just digital install media, same job a CD used to do

### 2. SSH — first attempt (and first faceplant)
- Initial SSH testing was done over a **NAT adapter** with port forwarding (host `22` → guest `22`), connecting to `localhost`
- `ssh antar@localhost` → connection aborted
- Turned out `openssh-server` was never actually installed. The Ubuntu installer had asked me during setup and I'd skipped right past it without noticing
- Fixed with the obvious sequence:
  ```bash
  sudo apt update
  sudo apt install openssh-server
  sudo systemctl enable ssh
  sudo systemctl start ssh
  sudo systemctl status ssh
  ```
- Reran the connection, accepted the TOFU (trust-on-first-use) host key prompt, and got in clean.
(prior notice: throughout the entire project, my username is referenced as "antar", like in ssh antar@localhost)
### 3. The Windows 11 VM (RIP)
- Configured with 3GB RAM / 2 CPU / 80GB storage → immediately hit `VERR_FILE_NOT_FOUND`, the `.vdi` had genuinely never been created
- Deleted and recreated the VM (4 CPU / 4GB RAM / 66GB storage) → this time it actually powered on, and immediately proved the RAM math above right. Too bad it wouldnt load the OS anyway. Only the VM ran, the OS didn't
- Decision made: drop the second guest, use the host as the second node instead (see "Why the plan changed")

### 4. Re-pointing SSH from NAT/localhost → Host-Only/static IP
This is the part that looked like a one-line change and was not:
- Switched the VM's network adapter from NAT to **Host-Only Adapter**
- `ip a` still showed a **dynamic** DHCP-assigned address (`192.168.56.102/24`), and not the static IP the setup actually needed
- Fixed via Netplan (`/etc/netplan/00-installer-config.yaml`): set `dhcp4: false`, `dhcp6: false`, added a static address under `enp0s3`
  ```bash
  sudo nano /etc/netplan/00-installer-config.yaml
  sudo netplan apply
  ```
- Config file: [`configs/netplan/00-installer-config.yaml`](configs/netplan/00-installer-config.yaml)
- Confirmed with `ip a`, confirmed host-to-VM reachability, and confirmed the real proof of a working link: a clean `ssh antar@192.168.56.10` session from the Windows host

**Practical upside worth calling out:** once this was live, I never needed to touch the VM through the VirtualBox console again for routine admin — it behaves like a normal remote server from here on. Ofcourse, as long as the Ubuntu Server remains On

Took a baseline snapshot here (`clean-server-network-labready`) before touching anything security-related.

### 5. Hardening pass

**a) Package updates**
```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt autoremove -y
sudo apt autoclean
```
Hit a pause mid-sequence: packages refusing to download. Root cause: the VM was Host-Only only at that point (great for host-VM SSH, but zero path for outbound access to the internet). Added a **second adapter** (NAT) alongside the Host-Only one so the VM could still reach the internet for updates while keeping the static Host-Only link for SSH. Reran the sequence clean.

**b) UFW (Uncomplicated Firewall)**
```bash
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```
Default deny incoming, allow outgoing, only `tcp/port22` explicitly opened.

**c) Listening ports checked**
```bash
ss -tulpn
```
Nothing concerning found in outputs `ModemManager` and `multipathd` were sitting there unused (irrelevant on a VM with no modem hardware / no enterprise multipath need) but doing no harm. Left them alone for this lab; flagged as a "strip these out for a tighter build" item for later.

**d) SSH hardening** (`/etc/ssh/sshd_config`, backed up first with `sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup`)

| Setting | Value | Why |
|---|---|---|
| `PermitRootLogin` | `no` | Blocks direct root SSH login; forces normal user + `sudo` |
| `PasswordAuthentication` | `yes` | Left on for now: SSH-key auth is a planned next step |
| `PubkeyAuthentication` | `yes` | Enabled ahead of time so I don't lock myself out once keys are set up |
| `PermitEmptyPasswords` | `no` | Blocks passwordless accounts from authenticating |
| `X11Forwarding` | `no` | Headless/terminal-only server, no GUI forwarding needed |
| `MaxAuthTries` | `3` | Disconnects after 3 failed attempts: basic anti-brute-force |
| `LoginGraceTime` | `50` | 50s to authenticate before the session expires |
| `MaxSessions` | `2` | Caps concurrent SSH sessions per connection |

Config file: [`configs/ssh/sshd_config`](configs/ssh/sshd_config)

**e) Fail2ban**

Installed and configured a dedicated `sshd` jail via a local override file (`jail.local`, not touching `jail.conf` directly — easier to maintain and roll back):

```ini
[DEFAULT]
bantime = 30m
findtime = 10m
maxretry = 3
backend = systemd

[sshd]
enabled = true
```

| Parameter | Purpose |
|---|---|
| `bantime = 30m` | Ban duration after repeated failures |
| `findtime = 10m` | Window in which failed attempts are counted |
| `maxretry = 3` | Failed attempts allowed before a ban |
| `backend = systemd` | Reads auth logs from `systemd-journald` |
| `[sshd] enabled = true` | Enables the jail for OpenSSH |

Hit a config error on first restart (`Failed to access socket path: /var/run/fail2ban/fail2ban.sock`). Root cause, found via `sudo fail2ban-client -t`: `jail.conf` already has an `[sshd]` section, and I'd pasted my config into the wrong (second) `[sshd]` block, causing a duplicate-section conflict. Restored from the backup, added the config to the correct section this time, config test passed, service came up active.

Config file: [`configs/fail2ban/jail.local`](configs/fail2ban/jail.local)

### 6. Post-hardening snapshot
Clean shutdown, second snapshot (`post-hardened-ubuntuserver`) — full rollback point with every security control in place, incase i intentionally decide to brick my own server out of frustration someday.

---

## Repo structure

```
virtual-home-lab/
├── README.md
├── LICENSE
├── .gitignore
├── docs/
│   └── network-diagram.png
├── configs/
│   ├── netplan/00-installer-config.yaml
│   ├── ssh/sshd_config
│   └── fail2ban/jail.local
└── screenshots/
    (numbered and described exports matching the writeup order: VM creation,
     port forwarding, SSH sessions, UFW status, fail2ban status, etc.)
```


## Lessons learned

Looking back, the actual friction never really came from the parts I expected it to. SSH itself wasn't the hard part, and Fail2ban wasn't the hard part either. The problems were smaller and dumber than that: an unchecked install checkbox, a mismatched network adapter, a config pasted into the wrong section. Real-world troubleshooting apparently runs almost entirely on stuff like that, and I got more practical hands-on networking/Linux experience out of this than out of any CompTIA A+ module.

**Skills covered:**
- VirtualBox networking and node setup (NAT vs. Host-Only, adapter stacking)
- Static IP configuration via Netplan
- SSH from the ground up (install → NAT/localhost test → Host-Only/static-IP production setup)
- UFW firewall basics
- `sshd_config` hardening
- Fail2ban jail setup end-to-end
- Snapshot/rollback discipline
- Debugging real config errors instead of assuming they won't happen

## Next steps
- SSH key-based auth, then flip `PasswordAuthentication` to `no`
- Strip `ModemManager`/`multipathd` for a tighter minimal build
- Automate the setup (user creation, static IP, UFW, SSH hardening) into a `setup.sh`. (Would save me atleast 48 hours fs)
- More RAM permitting, revisit the original two-VM design and finalize the original system back again

---

## License
MIT — see [`LICENSE`](LICENSE).
