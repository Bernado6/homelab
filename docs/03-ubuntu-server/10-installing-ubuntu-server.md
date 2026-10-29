# 10. Installing Ubuntu Server

This is the step where Windows disappears and a server appears. The installer is a text program that asks about a dozen questions, and almost all of them have a safe default. Two of them do not, and one of them (storage) is the kind of choice you only discover was wrong months later when the disk is mysteriously small. This page goes through every screen, and spends its background sections on the two that matter.

## What

Boot the OptiPlex from the USB stick made in [step 8](08-making-the-install-usb.md), walk through the Ubuntu Server installer, erase the SSD, and end with a machine that boots into a text login prompt with OpenSSH already running.

## Background: the boot menu, and what the entries mean

Pressing **F12** at the Dell logo opens the **One-Time Boot Menu**. Unlike F2, it changes nothing permanently; it just asks "which device, this once?" The list is built by the firmware from every device that contains something it recognises as bootable.

On a UEFI machine each entry is one of two things:

| Entry looks like | What it is |
|---|---|
| `UEFI: <something>` | A removable device with an EFI System Partition on it. The `<something>` is whatever name the USB stick's controller reports, which is often a generic string rather than the brand |
| `Windows Boot Manager` | Not a disk but a **boot entry** stored in the firmware, pointing at the Windows bootloader on the SSD. After the install this is replaced by an `ubuntu` entry |
| `SAMSUNG …` or another drive name | The internal SSD as a raw device. Booting it in legacy style; not what you want |

So the stick is the one with the `UEFI:` prefix, and the name after the prefix can be anything. Mine was **`UEFI: VendorCoProductCode 2.00`**, which is the stick's controller failing to report a real name. The firmware does not care; it found the `EFI` folder Rufus wrote in step 8, which is all it needs.

If the stick is not in the list at all, the firmware did not recognise it as bootable: see the Challenges table.

## Background: what the installer is, and the order it works in

Ubuntu Server's installer is called **Subiquity**. It is text-based (arrow keys, Tab, Space, Enter) and it runs entirely from the USB stick in RAM, which is why it can erase the SSD without harming itself. The ISO contains the installer and a compressed copy of the system; the installer copies that system onto the disk and then downloads updates for it if the network is up.

The screens come in a fixed order, and it helps to know that **nothing is written to the disk until the Storage confirmation screen**. Every screen before it can be revisited with **Back**. After it, the SSD is being erased.

```text
 Language ─► Keyboard ─► Install type ─► Network ─► Proxy ─► Mirror
                                                                │
                                              (nothing written yet)
                                                                ▼
                                            Storage ─► Confirm destructive action
                                                                │
                                                 (disk is being erased from here)
                                                                ▼
                     Profile ─► Ubuntu Pro ─► SSH ─► Snaps ─► Installing… ─► Reboot
```

The installer also talks to the network *during* setup: it fetches the package list from a mirror, checks for a newer installer version, and pulls security updates. That is why the Ethernet cable from [step 7](../02-hardware/07-display-keyboard-and-network.md) should be plugged in before you boot, and why the Mirror screen doubles as a connectivity test.

## Background: DHCP, and the address the Network screen shows

When the installer reaches the Network screen it has already asked the router for an address. The router runs a **DHCP** server (Dynamic Host Configuration Protocol): a device with no address broadcasts "anyone?", the router offers one from its pool, and the device accepts it. The result is a **lease**: an address that is yours for a fixed time (hours or days), renewed automatically while the machine is on.

The Network screen shows that lease next to the Ethernet interface, named something like `eno1`. Two things to take from it:

- It proves the cable and the router work.
- It is the address you will `ssh` to in [step 11](11-first-login-and-ssh.md), so write it down.

A lease is not permanent. If the machine is off when the lease expires, the router may hand the address to something else, and the server comes back with a different one. That is a real nuisance for a headless machine, and it is why [step 14](../05-networking/14-dhcp-reservation.md) pins the address in the router. For now, DHCP is correct; do not set a static address in the installer, because the router keeps ownership of the pool and a reservation is the cleaner fix.

## Background: what LVM is, and why the default leaves half the disk unused

This is the screen that catches people.

**LVM** (Logical Volume Manager) is a layer between the disk and the filesystem. Instead of carving the disk directly into partitions, you declare the disk a **physical volume**, pool it into a **volume group**, and then create **logical volumes** out of the pool that can be resized, snapshotted, or spread across disks later.

```text
 Without LVM                           With LVM
 ───────────                           ────────
 SSD 256GB                             SSD 256GB
 ├── EFI System Partition  (~1GB)      ├── EFI System Partition  (~1GB)
 └── /   ext4              (~255GB)    ├── /boot ext4           (~2GB)
                                       └── physical volume      (~253GB)
                                            └── volume group "ubuntu-vg"
                                                 └── logical volume "ubuntu-lv"  → /   ext4  100GB
                                                 └── (unallocated)                        ~153GB
```

Ubuntu's installer ticks **Set up this disk as an LVM group** by default, and when it does, it sizes the root logical volume at **about 100GB** and leaves the rest of the volume group unallocated. The reasoning is that a system administrator will come along later and create separate volumes for data, or use the free space for snapshots. For a homelab, the reality is that `df -h` shows a 100GB disk, you assume the seller lied about the 256GB, and the rest sits idle until you find out why.

| Option | What you get | Verdict |
|---|---|---|
| **LVM unticked** ✅ | One plain ext4 root partition using the whole disk | What I chose. Simple, the full 256GB is usable immediately, and `local-path` storage in K3s just uses space under `/` |
| LVM ticked (default) | A 100GB root volume and ~150GB sitting unallocated | Flexible, but only useful if you intend to manage volumes by hand. Fixable later with `lvextend` and `resize2fs` |
| Encrypted LVM | LVM plus full-disk encryption | Needs a passphrase typed at every boot, which a headless server cannot do without extra setup. Not for this machine. |

If you have already installed with LVM on, you do not need to reinstall: extend the logical volume into the free space (`sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv`) and then grow the filesystem (`sudo resize2fs /dev/ubuntu-vg/ubuntu-lv`). Both are safe on a mounted root filesystem. Unticking at install time is simply one fewer thing to remember.

## Background: the other choices, briefly

| Screen | What it is really asking |
|---|---|
| **Install type: minimized** | A smaller system with fewer command-line tools, intended for cloud images where a person never logs in. You *will* log in to this machine, so the standard install is right. |
| **Proxy** | Whether web traffic from this machine has to pass through a company proxy server. At home there is none, so blank. |
| **Mirror** | Which copy of Ubuntu's package archive to download from. Ubuntu runs mirrors worldwide and the installer picks a nearby one by country. The fact that it finds one at all proves the internet works. |
| **Ubuntu Pro** | Canonical's subscription for extended security updates and compliance tooling. Free for up to five personal machines. It can be attached any time later with `pro attach`, so skipping now loses nothing. |
| **OpenSSH server** | Installs and enables `sshd`, the program that answers `ssh` connections. Without it the machine is unreachable the moment the monitor is unplugged. The screen also offers to import public keys from a GitHub or Launchpad account; a convenience, not a requirement. |
| **Featured snaps** | Pre-packaged apps (Docker, MicroK8s, Nextcloud and so on) installed as **snaps**, Canonical's self-contained package format. Every one of these would compete with what K3s brings, and MicroK8s in particular is a different Kubernetes. None. |

## When

Immediately after [step 9](09-bios-settings.md): the same restart that saved the BIOS changes can go straight to F12. Plan for about half an hour with the monitor attached, which is the last time it is needed.

## How

### 1. Boot from the USB stick

Restart and tap **F12** for the boot menu.

Tap it from the moment the Dell logo appears. The menu lists everything bootable.

My flash drive showed as **`UEFI: VendorCoProductCode 2.00`**, a generic name rather than the brand. The other entries were `Windows Boot Manager` and the `SAMSUNG …` SSD; don't pick those.

Select the `UEFI:` entry with the arrow keys and press Enter. Expect a plain GRUB menu with **Try or Install Ubuntu Server** highlighted, then a scrolling wall of kernel messages, then the installer's first screen asking for a language.

My first attempt **went back to Windows**. See [Troubleshooting](../troubleshooting.md) for what to check.

If that happens to you, the firmware either did not find a bootloader on the stick or booted something else. The checks, in order:

1. Check the flash drive has Ubuntu files (`boot`, `casper`, `EFI`); if not, re-flash, trying **DD Image mode**.
2. Use a **rear** USB port.
3. Make sure you selected the USB entry.
4. Last resort: temporarily disable Secure Boot.

### 2. Start the installer

Choose **Try or Install Ubuntu Server**, then:

| Screen | What I chose | Notes |
|---|---|---|
| Language / keyboard | English / English (US) | |
| Install type | Ubuntu Server (**not** minimized) | |
| Network | DHCP | Note the IP shown |
| Proxy | **Blank** | Not needed at home |
| Mirror | Default | It picked `ke.archive.ubuntu.com` (Kenya), which also confirmed internet was working |
| Storage | Use entire disk → **256GB SSD**, **untick LVM** | Don't select the flash drive! With LVM on, Ubuntu only uses about 100GB by default |
| Profile | Name, server name, username, password | |
| Ubuntu Pro | **Skip for now** | Free for personal use; can enable later |
| SSH | ✅ **Install OpenSSH server** | Essential for working headless |
| Featured snaps | **None** | K3s brings its own components |

### 3. Screen by screen

What each screen looks like and what to expect, in the same order:

**Language and keyboard.** Arrow keys and Enter. The keyboard screen offers to detect the layout by asking you to press keys; choosing it from the list is faster.

**Install type.** Two radio buttons. Leave **Ubuntu Server** selected. The installer may also offer to update itself to a newer version here if the network is already up; accepting is fine and takes a minute.

**Network.** The Ethernet interface (`eno1` or similar) should already show `DHCPv4` and an address like `192.168.100.x`. If it shows no address, the cable or router is the problem; fix that before continuing, because the rest of the install needs the network. **Note the IP**: it is the one you will `ssh` to.

**Proxy.** Leave the field empty and continue.

**Mirror.** The installer proposes an archive URL and tests it. Seeing a mirror in your own country, `ke.archive.ubuntu.com` in my case, is the confirmation that DNS and the internet work from this machine.

**Storage.** The screen offers **Use an entire disk** with a drop-down of disks. Pick the **256GB SSD** (it shows the model name, SAMSUNG for me). The USB stick is also a disk and also appears in that list; picking it would install Ubuntu onto the stick you are booting from, or fail. Below the disk is the tick-box **Set up this disk as an LVM group**: **untick it**. Continue to a summary showing an EFI partition and a root partition using the whole disk, then **Done**, then a red confirmation that the disk will be erased. **Continue** here is the point of no return.

**Profile.** Four fields: your name, the **server's name** (its hostname; this is what later shows as the node name in `kubectl get nodes`, `quantum` in my case), a username, and a password typed twice. This user gets `sudo`. Pick a password you can type blind, because the login prompt will not show it.

**Ubuntu Pro.** **Skip for now**.

**SSH.** Tick **Install OpenSSH server**. Leave the key import alone unless you already have keys on GitHub and want them pre-installed.

**Featured snaps.** Scroll past everything, tick nothing, **Done**.

### 4. Wait, then reboot

The install took about 15–20 minutes. If you open **View full log**, a line ending in `SUCCESS` is good; press **Close** to go back. At **Reboot Now**, remove the USB when asked, then log in at the text prompt. The password stays invisible as you type.

The progress screen runs in two phases: copying the system (fast), then **downloading and installing security updates** from the mirror (depends on your connection). The **Reboot Now** button appears when both finish. On the reboot the machine goes through the Dell logo, a brief GRUB menu, more kernel messages, and stops at:

```text
Ubuntu 26.04.1 LTS <server name> tty1

<server name> login:
```

Type the username and Enter, then the password and Enter. Nothing echoes while you type the password, not even dots; that is normal on Linux. A prompt like `user@quantum:~$` means you are in. The next step moves from this keyboard to your laptop.

## Challenges and solutions

| Challenge | What was going on | Solution |
|---|---|---|
| F12 → USB entry, machine went back to Windows | The firmware did not boot the stick, or booted the SSD | 1) Check the flash drive has Ubuntu files (`boot`, `casper`, `EFI`); if not, re-flash, trying **DD Image mode**. 2) Use a **rear** USB port. 3) Make sure you selected the USB entry. 4) Last resort: temporarily disable Secure Boot. See [Troubleshooting](../troubleshooting.md) |
| Stick not in the F12 list at all | Not flashed as bootable, or a front port that the firmware initialises late | Same list: check the files, use a rear port |
| Installer can't see the SSD | BIOS → SATA Operation is still RAID On | Back to [step 9](09-bios-settings.md): **AHCI** |
| Ubuntu only uses ~100GB | The LVM default | Untick LVM during install, or extend the volume later with `lvextend` and `resize2fs` |
| Installer asks for a proxy | It always does | Leave it blank at home |
| Network screen shows no address | Cable not in, or in the router's WAN port, or the router's DHCP is off | Plug into a LAN port, check the link lights, go **Back** and forward again to retry |
| Mirror screen hangs or fails | No internet | Fix the network first; the install downloads updates and needs it |
| Picked the flash drive on the Storage screen | Both the SSD and the stick are "disks" | Check the size and model: 256GB SAMSUNG, not 8GB |
| Login fails after reboot | Typing the password in the username field, or Caps Lock | Username first, Enter, then password with nothing echoing |

## Lessons

> 💡 **What I learned:** the two screens that really matter are **Storage** (correct disk, LVM off) and **SSH** (on). Everything else can be left at the defaults.

> 💡 **Also:** the Mirror screen is a free network test. If it finds a mirror in your country, DHCP, DNS and the internet all work from the server, before you have typed a single command.

## Glossary

| Term | Meaning |
|---|---|
| One-Time Boot Menu (F12) | Dell's list of bootable devices for this boot only; nothing is changed permanently |
| `UEFI:` entry | A device with an EFI System Partition the firmware can boot; the name after it comes from the device and may be generic |
| Windows Boot Manager | The firmware boot entry pointing at Windows's bootloader; replaced by an `ubuntu` entry after install |
| Subiquity | The name of Ubuntu Server's text installer |
| GRUB | The bootloader that shows the "Try or Install" menu and later boots the installed system |
| DHCP | The router service that lends IP addresses from a pool |
| Lease | A DHCP address and the time it is valid for; not permanent until [step 14](../05-networking/14-dhcp-reservation.md) |
| `eno1` | The typical name Ubuntu gives the onboard Ethernet port |
| Mirror | One of many identical copies of Ubuntu's package archive; `ke.archive.ubuntu.com` is the Kenyan one |
| LVM | Logical Volume Manager: a layer that pools disks into a volume group and carves resizable logical volumes from it |
| Physical volume / volume group / logical volume | LVM's three layers: a disk or partition / a pool of them / a resizable slice of the pool that holds a filesystem |
| ext4 | The default Linux filesystem the root partition uses |
| EFI System Partition | The small FAT32 partition the firmware boots from; the installer creates it automatically |
| Minimized install | A stripped-down system meant for machines nobody logs in to; not this one |
| Ubuntu Pro | Canonical's subscription for extended updates; free for personal use, attachable later |
| OpenSSH server (`sshd`) | The service that answers `ssh` connections; the reason the monitor can be unplugged |
| Snap | Canonical's self-contained package format; the "featured snaps" screen offers apps packaged this way |
| Hostname | The server's name from the Profile screen; shows up as the node name in Kubernetes |
| tty1 | The first text console, where the login prompt appears on the attached monitor |

---

[◀ Previous: BIOS Settings](09-bios-settings.md) · [🏠 Home](../../README.md) · [Next: First Login and SSH ▶](11-first-login-and-ssh.md)
