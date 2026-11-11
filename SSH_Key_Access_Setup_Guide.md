# SSH Key Access: Windows 10 and WSL 2 (Arch Linux) to an Arch Linux PC

This guide records how SSH (Secure Shell) access was set up from a Windows 10 laptop and from Arch Linux running under WSL 2 (Windows Subsystem for Linux, version 2) on the same laptop, to an Arch Linux PC on the same home network. Logins use key pairs only, and a short host alias replaces the address and user name. An SSH agent holds the unlocked key, so the passphrase is not typed for every connection. Files are copied with `scp` (secure copy) in both directions.

## Hardware and Context

| Item | Detail |
|---|---|
| Server | Arch Linux PC, host name `<ARCH_HOST>`. Wired Ethernet (`eno1`), connected to the router through a pair of powerline adapters |
| Client 1 | Arch Linux under WSL 2 (distribution name `archlinux`) on the Windows 10 laptop |
| Client 2 | Windows 10 laptop (built-in OpenSSH client, used from PowerShell) |
| Network | One home network, router of the TP-Link AX1500 class at `<ROUTER_IP>`. Addresses are assigned by DHCP (Dynamic Host Configuration Protocol); no address reservation was made (see [Server Address](#server-address)) |
| Use | Mainly WSL → Arch PC. Copies are always started from the laptop, whether to or from the Arch PC |

Only the Arch PC runs an SSH server. Since every connection is started from the laptop, neither Windows nor WSL needs a server of its own. Windows and WSL exchange files directly through their shared file systems, without SSH (see [Copying Files](#copying-files)).

### Tested With

| Component | Version |
|---|---|
| Arch Linux (server PC) | Rolling release, tested November 2026; kernel 7.2.14-arch1-1 |
| OpenSSH (server PC) | 10.6p1 (Arch package openssh 10.6p1-2) |
| Windows 10 | 22H2 (OS build 19045.6466) |
| OpenSSH for Windows (built-in) | 9.5p1 (`OpenSSH_for_Windows_9.5p1, LibreSSL 3.8.2`) |
| WSL | 3.0.3.0; WSL kernel 6.18.54.1-1 |
| Arch Linux under WSL | Rolling release, with systemd enabled |
| OpenSSH (WSL) | 10.5p1 (Arch package openssh 10.5p1-1) |

## How It Works

**Key-based authentication.** Each client has its own key pair. The private key stays on the client and is protected by a passphrase. The public key is added to `~/.ssh/authorized_keys` on the Arch PC. Once both clients could log in with their keys, password logins were switched off on the server. A separate key per client means that either can be withdrawn by deleting one line from `authorized_keys`.

**Host key verification.** On the first connection the client shows the server's host key fingerprint and asks whether to trust it. The fingerprint was compared with the one displayed on the Arch PC itself, which confirms that the connection reached the right machine. The accepted key is stored in the client's `~/.ssh/known_hosts`.

**Host alias.** A block in each client's `~/.ssh/config` maps the short name `<ARCH_HOST>` to the address, user name and key. `ssh <ARCH_HOST>` and `scp file <ARCH_HOST>:` then need nothing further. `HostKeyAlias` makes the client file the host key under the alias rather than the IP address, so a change of address does not raise a new fingerprint prompt.

**SSH agent.** An agent holds the unlocked private key, and `ssh` and `scp` ask it for signatures instead of prompting for the passphrase. The two clients behave differently:

| Client | Agent | Passphrase asked |
|---|---|---|
| WSL | `ssh-agent`, started on demand by systemd; the key is added on first use (`AddKeysToAgent yes`) | Once after each start of WSL; the agent is empty after `wsl --shutdown` |
| Windows | Built-in `ssh-agent` service; the key is added once with `ssh-add` | Once only. The Windows agent stores added keys protected under the Windows account, and they remain after a reboot (documented behaviour; persistence across a reboot was not tested in this setup) |

**Why WSL needs no special network set-up.** On Windows 10, WSL 2 sits behind NAT (network address translation) on a virtual network inside Windows. Outbound connections from WSL to the LAN (local area network) pass through Windows without configuration. Inbound connections to WSL from the LAN would need port forwarding, which this setup avoids because WSL is only ever a client. The alternative "mirrored" networking mode, which removes the NAT, requires Windows 11 22H2 or later.

## Arch Linux PC (Server)

### Software

- `openssh` (from the Arch repositories: `pacman -S openssh`; it was already installed on this PC). Provides the server, `sshd`.

### Checking the Starting State

```bash
systemctl is-enabled sshd                               # was: disabled
systemctl is-active sshd                                # was: inactive
systemctl is-active firewalld ufw nftables iptables     # all: inactive
```

No firewall was running (Arch Linux enables none by default), so no port had to be opened.

### Enabling the Server

```bash
sudo systemctl enable --now sshd
systemctl is-active sshd               # active
sudo ss -tlnp 'sport = :22'            # sshd listening on 0.0.0.0:22 and [::]:22
```

`enable --now` starts the server immediately and at every boot.

### Host Key Fingerprint

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
# 256 SHA256:<HOST_KEY_FINGERPRINT> root@<ARCH_HOST> (ED25519)
```

This value is compared with the fingerprint shown by each client on its first connection.

### Key-Only Logins

This step was carried out **after** both clients' keys had been installed and tested. Applied earlier, it would lock out the clients that still rely on a password.

The settings go in a drop-in file rather than in `/etc/ssh/sshd_config`, so package updates do not overwrite them. Arch's `sshd_config` includes the drop-in directory on its second line, and `sshd` keeps the first value it reads for each setting, so the drop-in takes precedence:

```bash
grep -n '^Include' /etc/ssh/sshd_config
# 2:Include /etc/ssh/sshd_config.d/*.conf

sudo tee /etc/ssh/sshd_config.d/10-key-only.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
EOF

sudo sshd -t && echo "syntax OK"          # checks the configuration
sudo sshd -T | grep -Ei '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
# all three: no

sudo systemctl reload sshd                # applies it; open sessions are kept
```

The result was verified from WSL:

```bash
ssh -o PubkeyAuthentication=no <ARCH_USER>@<ARCH_IP>
# <ARCH_USER>@<ARCH_IP>: Permission denied (publickey).     (no password prompt is offered)
```

### Server Address

The Arch PC receives its address from the router by DHCP (`ip route` shows `proto dhcp`). A DHCP reservation on the router would fix it, but it could not be made: login to the router failed, and an attempt at a factory reset did not take effect (see [Troubleshooting Notes](#troubleshooting-notes)). The address was therefore left dynamic. Home routers commonly give a device the same address again, and the PC kept the same address throughout this setup, but this is not guaranteed.

If the Arch PC stops answering, its current address is shown on the PC by:

```bash
ip -brief address show eno1
```

Only the `HostName` line of the alias on each client then needs changing. Owing to `HostKeyAlias`, no new fingerprint prompt is expected (this case has not yet arisen, so it is untested).

For a later attempt at the reservation: TP-Link's instructions for the Archer AX10 and AX1500 place it under **Advanced > Network > DHCP Server > Address Reservation > Add**, where the PC's MAC (Media Access Control) address, shown by `ip link show eno1` as `link/ether <ARCH_MAC>`, is paired with the address to keep. The reserved address has to lie within the router's DHCP address pool.

## WSL Side (Arch Linux under WSL 2)

WSL's default distribution on this laptop is Ubuntu, so the Arch distribution is opened with `wsl -d archlinux`. The commands below are run in that shell.

### Software

```bash
sudo pacman -S openssh       # was not installed in WSL
```

### First Connection

```bash
ssh <ARCH_USER>@<ARCH_IP> uname -nr
```

The fingerprint shown was compared with the server's (see [Host Key Fingerprint](#host-key-fingerprint)) and accepted with `yes`; the password was then entered. Output: `<ARCH_HOST> 7.2.14-arch1-1`.

### Key Pair

```bash
ssh-keygen -t ed25519 -C "wsl-laptop"      # default location; a passphrase was set
ssh-copy-id <ARCH_USER>@<ARCH_IP>          # asks for the server password one last time
```

`ssh-keygen` creates `~/.ssh/id_ed25519` (private) and `~/.ssh/id_ed25519.pub` (public). `ssh-copy-id` appends the public key to `~/.ssh/authorized_keys` on the server and reports `Number of key(s) added: 1`.

A key-only test proves that the key, not the password, is used:

```bash
ssh -o PasswordAuthentication=no -o KbdInteractiveAuthentication=no <ARCH_USER>@<ARCH_IP> uname -nr
# Enter passphrase for key '/home/<WSL_USER>/.ssh/id_ed25519':
# <ARCH_HOST> 7.2.14-arch1-1
```

### Host Alias

`~/.ssh/config`:

```
Host <ARCH_HOST>
    HostName <ARCH_IP>
    User <ARCH_USER>
    IdentityFile ~/.ssh/id_ed25519
    HostKeyAlias <ARCH_HOST>

Host *
    AddKeysToAgent yes
```

```bash
chmod 600 ~/.ssh/config
ssh <ARCH_HOST> uname -nr
```

The first connection under the alias shows the fingerprint once more, because the key is now filed under the alias. The prompt also reports that the key is already known under the IP address. Specific `Host` blocks come before `Host *`, since SSH uses the first value it finds for each setting.

### Agent

The Arch openssh package provides a systemd user service, `ssh-agent.service`, and a matching socket, `ssh-agent.socket`. With the socket enabled, systemd creates the agent's socket at the start of the session and starts the agent only when it is first contacted. This requires systemd to be running inside WSL; on this system `/etc/wsl.conf` contains `systemd=true` under `[boot]`, and `ps -p 1 -o comm=` reports `systemd`.

```bash
systemctl --user enable --now ssh-agent.socket
systemctl --user cat ssh-agent.socket | grep -i listen    # ListenStream=%t/ssh-agent.socket
echo "$XDG_RUNTIME_DIR"                                    # /run/user/1000
```

`%t` stands for `$XDG_RUNTIME_DIR`. The shell is told where the agent is by one line in `~/.bashrc`:

```bash
echo 'export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/ssh-agent.socket"' >> ~/.bashrc
source ~/.bashrc
```

With `AddKeysToAgent yes` in `~/.ssh/config` (above), the first connection asks for the passphrase and hands the key to the agent; later connections ask nothing:

```bash
ssh <ARCH_HOST> uname -nr       # passphrase asked
ssh <ARCH_HOST> uname -nr       # no prompt
ssh-add -l                      # 256 SHA256:<WSL_KEY_FINGERPRINT> wsl-laptop (ED25519)
```

The arrangement was checked after a restart of WSL (`wsl --shutdown` in PowerShell, then `wsl -d archlinux`): `SSH_AUTH_SOCK` was set, `ssh-add -l` reported `The agent has no identities.`, the first connection asked for the passphrase and the second did not.

## Windows Side (Windows 10 Laptop)

The commands are run in **PowerShell**. The Command Prompt (`cmd.exe`) does not treat single quotes as quoting characters, which breaks remote commands written in the Linux style (see [Troubleshooting Notes](#troubleshooting-notes)).

### Software

The OpenSSH client built into Windows 10 was used, at `C:\Windows\System32\OpenSSH\`; `ssh -V` reports `OpenSSH_for_Windows_9.5p1`. No further installation was needed.

### Key Pair

```powershell
ssh-keygen -t ed25519 -C "windows-laptop"       # default location; a passphrase was set
```

The key is saved as `C:\Users\<WINDOWS_USER>\.ssh\id_ed25519` and `id_ed25519.pub`.

Windows has no `ssh-copy-id`. The public key was therefore copied with `scp` and appended on the server (two password prompts, the last for this client):

```powershell
scp "$env:USERPROFILE\.ssh\id_ed25519.pub" <ARCH_USER>@<ARCH_IP>:/tmp/windows-laptop.pub
ssh <ARCH_USER>@<ARCH_IP> "cat /tmp/windows-laptop.pub >> ~/.ssh/authorized_keys && rm /tmp/windows-laptop.pub"
ssh -o PasswordAuthentication=no -o KbdInteractiveAuthentication=no <ARCH_USER>@<ARCH_IP> uname -nr
```

The last command succeeded after the passphrase prompt, which shows that the key was installed.

### Host Alias

```powershell
@"
Host <ARCH_HOST>
    HostName <ARCH_IP>
    User <ARCH_USER>
    IdentityFile ~/.ssh/id_ed25519
    HostKeyAlias <ARCH_HOST>
"@ | Add-Content -Encoding ascii "$env:USERPROFILE\.ssh\config"

ssh <ARCH_HOST> uname -nr
```

`-Encoding ascii` is a precaution: in Windows PowerShell the `>` redirection writes UTF-16 text, which OpenSSH may fail to read. `~` in `IdentityFile` is understood by Windows OpenSSH. As in WSL, the first connection under the alias shows the fingerprint once.

### Agent

The `ssh-agent` service belongs to the same built-in installation (`C:\Windows\System32\OpenSSH\ssh-agent.exe`) but is disabled by default.

In PowerShell **run as administrator**:

```powershell
Set-Service ssh-agent -StartupType Automatic
Start-Service ssh-agent
Get-Service ssh-agent | Select-Object Name, Status, StartType    # Running, Automatic
```

In an ordinary PowerShell window:

```powershell
ssh-add "$env:USERPROFILE\.ssh\id_ed25519"      # passphrase asked once
ssh-add -l                                      # 256 SHA256:<WINDOWS_KEY_FINGERPRINT> windows-laptop (ED25519)
ssh <ARCH_HOST> uname -nr                       # no prompt
```

The Windows agent keeps added keys in the registry, protected with the Windows account's credentials, so the key is in effect unlocked whenever that account is logged in. `ssh-add -d "$env:USERPROFILE\.ssh\id_ed25519"` removes it from the agent.

## Copying Files

From WSL or Windows, with the alias:

```bash
scp report.pdf <ARCH_HOST>:~/Documents/        # laptop → Arch PC
scp <ARCH_HOST>:~/Documents/report.pdf .       # Arch PC → current folder on the laptop
```

`ssh <ARCH_HOST>` with no command opens an interactive shell on the Arch PC; `exit` or Ctrl+D returns to the laptop.

Between Windows and WSL no SSH is used:

| From | Path to the other side |
|---|---|
| WSL | Windows drives under `/mnt/c/...` |
| Windows | WSL files under `\\wsl$\archlinux\...` (for example, in File Explorer) |

## Troubleshooting Notes

Issues encountered and how they were resolved during setup:

1. **`bash: line 1: hostname; uname -r: command not found`** – The command `ssh user@host 'hostname; uname -r'` had been typed in the Windows Command Prompt. `cmd.exe` passed the single quotes on literally, so the server's shell took the whole quoted string as one command name. Resolved by using PowerShell on Windows (or quoting with double quotes), and by using `uname -nr`, which needs no quoting at all.
2. **Router login failed; factory reset did not take effect** – The credentials on the router's label and a password believed to be current were both refused. The Reset button was then held for about 30 seconds, yet the router kept its settings: the login page still asked for an existing password and the Wi-Fi name was unchanged. On TP-Link routers of this class there is no factory login password; after a completed reset the management page asks for a **new** password to be created, and the password printed on the label is the Wi-Fi key. TP-Link's guide says the Reset button is to be held until all LEDs turn on. The cause of the failed reset was not established, and the attempt was abandoned with 7 of the router's 10 login attempts used. The gateway was confirmed to be a TP-Link device from its MAC address prefix (`ip neigh show <ROUTER_IP>`). A completed reset would also erase the Wi-Fi and internet settings.
3. **Fingerprint prompt after changing the alias** – Expected. With `HostKeyAlias`, the host key is filed under the alias, so a new or renamed alias brings one fingerprint prompt (compare it with the server's value). Entries under an old alias are removed with `ssh-keygen -R <OLD_ALIAS>`, which keeps a backup as `known_hosts.old`.

## Quick Reference

| Action | Where | How |
|---|---|---|
| Open a shell on the Arch PC | WSL or Windows | `ssh <ARCH_HOST>` |
| Copy to / from the Arch PC | WSL or Windows | `scp <FILE> <ARCH_HOST>:<PATH>` / `scp <ARCH_HOST>:<PATH> .` |
| Show the server's host key fingerprint | Arch PC | `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` |
| Find the Arch PC's current address | Arch PC | `ip -brief address show eno1` |
| Update the address after a change | WSL / Windows | Edit `HostName` in `~/.ssh/config` / `%USERPROFILE%\.ssh\config` |
| Check server state | Arch PC | `systemctl is-active sshd`; `sudo sshd -T \| grep -i authentication` |
| Server key-only settings | Arch PC | `/etc/ssh/sshd_config.d/10-key-only.conf`, then `sudo systemctl reload sshd` |
| Authorise or withdraw a client key | Arch PC | Add or delete its line in `~/.ssh/authorized_keys` |
| List keys held by the agent | WSL or Windows | `ssh-add -l` |
| Open the Arch distribution in WSL | Windows | `wsl -d archlinux` |
| Test reachability of the server | Windows | `Test-NetConnection <ARCH_IP> -Port 22` |

## Sources

- OpenSSH manual pages: `sshd_config(5)`, `ssh_config(5)`, `ssh-keygen(1)`, `ssh-copy-id(1)`, `ssh-agent(1)`, `ssh-add(1)` (`man <page>`, or https://man.openbsd.org/)
- Arch Wiki, OpenSSH: https://wiki.archlinux.org/title/OpenSSH and SSH keys: https://wiki.archlinux.org/title/SSH_keys
- Microsoft, OpenSSH key management (Windows agent): https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement
- Storage of keys added to the Windows agent (registry, protected with DPAPI, the Windows Data Protection API): https://blog.ropnop.com/extracting-ssh-private-keys-from-windows-10-ssh-agent
- Microsoft, Accessing network applications with WSL (NAT and mirrored modes): https://learn.microsoft.com/windows/wsl/networking
- TP-Link Community, Address Reservation (Advanced > Network > DHCP Server): https://community.tp-link.com/en/home/forum/topic/197552 and, for the Archer AX10, https://community.tp-link.com/en/home/forum/topic/847400
- TP-Link, Archer AX10 & AX1500 User Guide, FAQ (login password and reset): https://www.tp-link.com/us/user-guides/archer-ax10&ax1500_v1/faq
