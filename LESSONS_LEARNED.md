# Lessons Learned

## Live sessions aren't "installed" just because they boot

I burned time troubleshooting why every change vanished on restart before realizing the VM was never actually installed — it was booting the live ISO every single time because "Connect at power on" was still checked. A good reminder to verify the fundamentals (is this actually persistent storage?) before debugging anything more specific.

## Root-everything is convenient and also a little scary

Building the Samba share around the root account got things working fast, but it made me actually think about blast radius for the first time — anyone with those credentials gets full read/write to that folder. Fine for a single-user personal VM, but it clarified why real environments use scoped service accounts instead of just reaching for root.

## Samba has its own password system, and that trips people up

`smbpasswd` sets a credential completely separate from the Linux login password, which isn't obvious until you try logging into the share with your normal password and it fails. Small thing, but worth internalizing since it's an easy source of "why isn't this working" moments.

## Two permission layers means two places to check

Samba's `read only` setting and the Linux filesystem's actual permissions are independent of each other. If they disagree, the more restrictive one usually wins in confusing ways. Before assuming a share is misconfigured, I now check both layers instead of just the `smb.conf` file.

## Planning the AD DC phase before touching anything was worth it

Writing out the domain name, realm, hostname, and static IP decisions ahead of time — and specifically flagging the static IP change as the riskiest step — made it obvious where things could go wrong (like conflicting with a DHCP-assigned address) before I actually run the commands. Cheap insurance for something that could otherwise take the VM off the network.

## What I'd do differently next time

- Check VM boot/media settings first, before assuming a persistence bug is something more complex
- Set up a non-root user for services like Samba from the start, rather than defaulting to root and revisiting it later
- Run `ip route` and confirm the DHCP range *before* deciding on a static IP, not as an afterthought right before applying it
