# Home Lab — Linux Mint VM + Samba File Server (with AD DC planned)

## Environment

- **Hypervisor**: VMware
- **Guest OS**: Linux Mint
- **VM IP**: 10.0.0.91 (DHCP-assigned initially)
- **Interface**: ens33
- **Working user**: root

## Part 1 — Fixing a Persistence Issue

Ran into an issue where every VM restart wiped all changes. Root cause: the VM's CD/DVD device had "Connect at power on" checked, so it kept booting from the Linux Mint ISO instead of the installed OS — meaning every session was a non-persistent live boot.

**Fix:**
1. Booted the live session one more time
2. Ran the "Install Linux Mint" installer, choosing "Erase disk and install" (scoped to the VM's virtual disk only)
3. Unchecked "Connect at power on" for the CD/DVD device in VM Settings
4. From then on, the VM booted from its own virtual disk and persisted changes normally

## Part 2 — Samba File Server Setup

**Install:**
```bash
sudo apt update
sudo apt install samba
```

**Shared folder** (working as root, so it lives under `/root/shared` rather than a home directory):
```bash
mkdir -p /root/shared
```

**Share config** — added to `/etc/samba/smb.conf`:
```ini
[Shared]
   path = /root/shared
   available = yes
   valid users = root
   read only = No
   browsable = yes
   public = Yes
   writable = No
   admin users = root
```

Key details:
- `valid users = root` restricts the share to the root account, authenticated via a separate Samba password
- `admin users = root` is required because Samba blocks root logins by default even with correct credentials
- `read only` / `writable` control client-side modification rights — for a strictly read-only share later, `admin users` would need to be removed too, since admin users bypass read-only

**Set Samba password and apply:**
```bash
sudo smbpasswd -a root
sudo testparm              # validate config syntax
sudo systemctl restart smbd
sudo ufw allow samba       # only if ufw is active
```

**Connect from Windows:**
```
\\10.0.0.91\Shared
```
Authenticated with the `root` Samba password (separate from the Linux login password).

## Part 3 — Permissions Model

Two independent permission layers exist: Samba-level (`read only` in `smb.conf`) and Linux filesystem-level (`chmod`/`chown`). For a genuinely read-only share, both need to agree.

**Security note:** since the share runs under root, anyone who authenticates gets full read/write access to `/root/shared` — scoped only to that folder, no shell access, but still a broader blast radius than ideal for anything beyond a personal single-user VM. A dedicated non-root user is the better long-term setup.

## Part 4 — Clipboard Sharing (VMware side-quest)

Fixed copy/paste between the Windows host and Mint guest:
- Enabled "copy and paste" and "drag and drop" under VM Settings → Options → Guest Isolation (must shut down the VM first — greyed out while running; if still stuck, check the `.vmx` file for `isolation.tools.copy.disable` / `isolation.tools.paste.disable` set to `TRUE`)
- Installed VMware Tools integration:
```bash
sudo apt update
sudo apt install open-vm-tools open-vm-tools-desktop
sudo reboot
```

## Part 5 — Next Phase (Planned): Samba as an AD Domain Controller

**Goal:** turn this VM into a full Active Directory Domain Controller so Windows Pro/Enterprise/Education machines can domain-join and be centrally managed (Home edition can't join a domain).

**Why Samba AD DC over FreeIPA/OpenLDAP:** only Samba's AD DC mode supports genuine Windows domain-join with centralized login and group policy — FreeIPA/OpenLDAP handle Linux clients well but Windows can't domain-join to them.

**Decided so far:**
- Domain: `home.lan` / Realm: `HOME.LAN`
- DC hostname: `dc1.home.lan`
- Static IP planned: `10.0.0.91` (currently still DHCP)

**Planned steps** (not yet executed):
1. Stop/disable old Samba file-server services, back up `smb.conf`
2. Install `krb5-config krb5-user winbind libpam-winbind libnss-winbind`
3. Convert to a static IP via Netplan (verify real gateway with `ip route` first — flagged as the riskiest step, since a wrong static IP could conflict with DHCP)
4. Set hostname to `dc1.home.lan` and update `/etc/hosts`
5. Provision the domain with `samba-tool domain provision --use-rfc2307 --interactive`
6. Swap in the generated Kerberos config
7. Start and enable the `samba-ad-dc` service
8. Verify via `samba-tool domain level show` and SRV record lookups for LDAP/Kerberos

This part hasn't been executed yet — will follow up with a dedicated write-up once it's done.
