# 8. Making the Ubuntu Install USB

This is the first step that changes something, and it is easy to get wrong in ways that only show up two steps later, when the machine refuses to boot from the stick. Most of the page is about what a bootable USB actually is, because once that is clear every option in Rufus explains itself.

## What

Turn a downloaded Ubuntu Server file (an **ISO**) into a **bootable USB flash drive** that the OptiPlex can start from, so that in [step 10](10-installing-ubuntu-server.md) the installer can wipe the internal SSD and put Ubuntu on it.

**You need a USB flash drive** (4GB+). I bought an **8GB** one.

Things that **don't** work as a boot drive:
- ❌ Saving the ISO on the same SSD you'll install onto (the installer would wipe itself)
- ❌ An Android phone (needs root plus special apps)
- ❌ A USB-C cable between two computers (that doesn't create a bootable drive)

| Item | What I used |
|---|---|
| OS | **Ubuntu Server 26.04.1 LTS**, Intel/AMD 64-bit (~3GB), from ubuntu.com/download/server |
| Flashing tool | **Rufus 4.15 Portable** (`rufus-4.15p.exe`, Windows x64), from rufus.ie |
| Where | On the OptiPlex itself, while Windows was still installed |

## Background: what an ISO is, and why copying it is not enough

An **ISO** is a single file that contains an exact image of a disc: every file, the folder structure, and the boot sector that tells a machine "start here". The name comes from ISO 9660, the standard for CD-ROM filesystems, and the format stuck even though nobody has used a CD for this in years. The Ubuntu Server download is one of these, about 3GB.

The important point is that an ISO is a picture of a *disk*, not a file to put *on* a disk. If you drag `ubuntu-26.04.1-live-server-amd64.iso` onto a flash drive in Windows Explorer, you get a flash drive with one large file on it. The firmware looks at that drive, finds no boot sector, no EFI folder, nothing it recognises, and skips it.

```text
 Copy the file                          Flash the image
 ─────────────                          ───────────────
 USB drive                              USB drive
 └── ubuntu-26.04.1-….iso   ✗           ├── EFI/                ✓ firmware finds the bootloader here
                                        ├── boot/
                                        ├── casper/             ← the installer's compressed filesystem
                                        ├── pool/
                                        └── …
```

A flashing tool like Rufus *unpacks the picture onto the drive*: it writes a partition table, creates a filesystem, copies the boot files and the installer where the firmware expects them, and marks the drive bootable. That is the difference between "a drive with an ISO on it" and "a bootable drive".

### The two things the firmware needs to find

When you later press F12 and pick the USB stick, the machine's firmware (the UEFI you will meet in [step 9](09-bios-settings.md)) does two things:

1. Reads the **partition table** at the start of the drive to find out what partitions exist.
2. Looks for a partition it can boot from, and inside it a bootloader file at a known path (`EFI/BOOT/BOOTX64.EFI` on a 64-bit PC).

Rufus has one setting for each of those, which is why the two choices in the How are **Partition scheme** and **Target system**.

## Background: GPT vs MBR, UEFI vs legacy BIOS

These two pairs are the source of almost every "it won't boot" story. They are two separate choices that have to agree with each other and with the machine.

### The partition table: MBR or GPT

| | MBR (Master Boot Record) | GPT (GUID Partition Table) |
|---|---|---|
| Age | 1983 | 2000s, part of the UEFI specification |
| Max disk size | 2TB | effectively unlimited |
| Max partitions | 4 primary | 128 |
| Where boot code lives | In the first sector of the disk | In a dedicated **EFI System Partition** |
| Pairs naturally with | Legacy BIOS | UEFI |

### The firmware: legacy BIOS or UEFI

The **firmware** is the small program on the motherboard that runs before any operating system. Its job on power-on is to find a disk to boot from and hand over.

| | Legacy BIOS | UEFI |
|---|---|---|
| How it boots | Jumps blindly to the first sector of the disk and runs whatever is there | Reads the partition table, finds the EFI System Partition, runs a named `.EFI` file |
| Disk format it expects | MBR | GPT |
| Secure Boot | Not possible | Available |
| On the OptiPlex 7070 | Available only as a compatibility mode called **CSM** | The native mode, and the default |

**CSM** (Compatibility Support Module) is a UEFI firmware pretending to be a legacy BIOS so that old MBR-only operating systems still boot. Modern Linux does not need it. Rufus's option **UEFI (non CSM)** means "build this drive for a real UEFI boot, do not bother with the legacy pretend-mode".

Why GPT + UEFI and not the other pair, for this machine:

| Combination | Verdict |
|---|---|
| **GPT + UEFI (non CSM)** ✅ | Matches how the OptiPlex boots out of the box, works with Secure Boot left on, and the SSD Ubuntu creates will be GPT too. What I used. |
| MBR + BIOS or UEFI-CSM | Would boot, but only after turning on CSM in the firmware and usually turning Secure Boot off. Extra steps for no benefit. |
| MBR + UEFI | Rufus offers it for odd firmware. Not needed here. |

Rufus reads the ISO and defaults to the sensible pair, so the setting is mostly a check that it chose what you expected.

### Secure Boot, briefly

**Secure Boot** is a UEFI feature: the firmware only runs bootloaders signed by a key it trusts, which blocks malware that hides in the boot path. Ubuntu's bootloader (`shim`) is signed with a Microsoft key that every PC firmware ships with, so Ubuntu boots with Secure Boot left on. That is why [step 9](09-bios-settings.md) leaves it on, and why "disable Secure Boot" is the last resort in the troubleshooting list rather than the first.

## Background: ISO mode vs DD mode in Rufus

When you press START, Rufus asks how to write the image. The two modes build the stick in different ways:

| Mode | What Rufus does | Result |
|---|---|---|
| **ISO Image mode** (recommended, and what I used) | Creates a normal FAT32 partition and copies the files from the ISO into it | A drive Windows can still open and show files on. Easy to inspect: you should see `boot`, `casper` and `EFI` folders. |
| **DD Image mode** | Copies the ISO byte for byte onto the drive, exactly as `dd` would on Linux | A perfect clone of the image. Windows may complain the drive needs formatting afterwards, which is normal. Some firmware boots this when ISO mode fails. |

Ubuntu's ISOs are built as **hybrid images**, meaning they work either way. ISO mode is the default because it is the one you can look at afterwards. If the machine will not boot the stick in [step 10](10-installing-ubuntu-server.md), re-flashing in DD mode is the first thing to try.

## Why a separate drive, and why Rufus

### Why the ISO cannot live on the target SSD

The installer runs *from* the boot medium and *writes to* the target disk. If both are the same SSD, the moment the installer formats the disk it deletes itself. The USB stick exists so that the thing doing the installing is somewhere the installer will never touch. This is also why the Storage screen in step 10 carries a warning not to pick the flash drive as the target.

### Why Rufus and not the alternatives

| Tool | Runs on | Verdict |
|---|---|---|
| **Rufus** ✅ | Windows | Small, portable (no install), shows the partition scheme and target system explicitly so you learn what you are choosing. Windows was still on the OptiPlex, so it was the natural pick. |
| balenaEtcher | Windows, macOS, Linux | Simpler, three clicks, hides every choice. Always writes in DD style. Fine if you are flashing from a Mac. |
| Ventoy | Windows, Linux | You copy many ISOs onto one stick and pick at boot. Great once you own several ISOs; overkill for one. |
| `dd` | macOS, Linux | Built in, no download, and one wrong device name wipes the wrong disk. Only if you are already comfortable with it. |
| Ubuntu's Startup Disk Creator | Ubuntu Desktop | Needs an existing Ubuntu machine, which I did not have yet. |

Rufus is a Windows program, so it ran on the OptiPlex in the Windows that came with it. That is the slightly surprising part of this step: the machine makes its own replacement.

## When

After [step 7](../02-hardware/07-display-keyboard-and-network.md), because you need the monitor, keyboard and a working internet connection to download 3GB. Before [step 9](09-bios-settings.md): the BIOS changes and the first boot from USB come straight after this, so have the stick finished first.

## Steps

### 1. Download the Ubuntu Server ISO

1. Download the Ubuntu Server ISO to **Downloads** on the OptiPlex, *not* onto the flash drive.

Go to ubuntu.com/download/server and take the **Ubuntu Server** LTS download for **Intel/AMD 64-bit** (also written `amd64` or `x86_64`). Three things to check on the download page before clicking:

| Check | Right | Wrong |
|---|---|---|
| Edition | **Server** (text installer, no desktop) | Desktop (a GUI you would never use headless, and a bigger image) |
| Architecture | **amd64** (Intel and AMD PCs, including the OptiPlex) | arm64 (Raspberry Pi and similar) |
| Release | An **LTS** release, for five years of updates | An interim release with nine months of support |

The file is about 3GB and the name will look like `ubuntu-26.04.1-live-server-amd64.iso`. Let it finish; a half-downloaded ISO flashes fine and then fails in strange ways.

### 2. Download Rufus

2. Download `rufus-4.15p.exe`.
   - Chrome blocked it as an "uncommon download". I kept it via `Ctrl + J` → **Keep**.
   - To check it's genuine: right-click → **Properties → Digital Signatures**, which should show **Akeo Consulting**.

The `p` in the filename means **portable**: it is a single `.exe` that runs without installing anything, which suits a machine that is about to be wiped. Chrome's warning is about the file being uncommon, not about it being known as malicious; the digital signature is the real check. Akeo Consulting is the company behind Rufus, so a signature from anyone else means you have the wrong file.

### 3. Set up Rufus

3. Open Rufus with the flash drive plugged in:
   - **Device:** the flash drive (check the size!)
   - **Boot selection:** the Ubuntu ISO
   - **Partition scheme:** GPT
   - **Target system:** UEFI (non CSM)

Field by field:

| Field | Set to | Why, and what to watch |
|---|---|---|
| **Device** | The flash drive | Rufus lists removable drives. Check the **size** matches your stick (an 8GB stick shows as roughly 7.5GB). Whatever is here gets erased completely. |
| **Boot selection** | The Ubuntu ISO, via **SELECT** | Browse to Downloads and pick the `.iso`. Rufus reads it and fills in sensible defaults below. |
| **Partition scheme** | **GPT** | The partition table format the UEFI firmware expects. See Background. |
| **Target system** | **UEFI (non CSM)** | Build the stick for a native UEFI boot, no legacy compatibility. Rufus fills this in to match GPT. |
| Volume label, file system, cluster size | Leave the defaults | Rufus names the volume after the ISO and uses FAT32, which is what UEFI firmware reads. |

If the Device box is empty, the stick is not being detected: try another port, or check that Windows shows it in Explorer.

### 4. Write it

4. **START** → *Write in ISO Image mode* → wait for **READY**.

Rufus warns that all data on the device will be destroyed, which is the point. Choose **Write in ISO Image mode** when asked (see the Background on modes for when DD is the better pick). Writing 3GB to a flash drive takes a few minutes; the green bar reaching **READY** means it is done. Eject the stick properly from the Windows tray before pulling it out, so the last writes are flushed.

### 5. Check the stick before you need it

Open the flash drive in Windows Explorer. In ISO Image mode you should see folders including `boot`, `casper` and `EFI`. If you see a single `.iso` file instead, the image was copied rather than flashed; go back to step 3. This thirty-second look is the same check the troubleshooting list asks for when the boot fails in step 10, so do it now while the fix is one click away.

## Challenges and solutions

Nothing failed for me here beyond the Chrome warning. The table also lists the traps this page is designed to avoid, because they surface later as boot failures.

| Challenge | What was going on | Solution |
|---|---|---|
| Chrome blocked `rufus-4.15p.exe` as an "uncommon download" | Chrome flags rarely downloaded executables regardless of whether they are safe | `Ctrl + J` → **Keep**. Then confirm the signature: right-click → Properties → Digital Signatures → **Akeo Consulting** |
| Flash drive has one `.iso` file on it and the machine will not boot from it | The ISO was copied, not flashed | Use Rufus (or any flashing tool); the stick must end up with `boot`, `casper` and `EFI` folders |
| Picked the wrong Device in Rufus | Rufus erases whatever is selected | Check the size before START. On a laptop with an external drive plugged in, unplug everything except the target stick |
| Downloaded Ubuntu **Desktop** or the **arm64** image | Wrong edition or architecture for a headless Intel server | Re-download: **Server**, **amd64** |
| Stick boots on one machine but not another | Partition scheme or target system does not match that machine's firmware | For the OptiPlex: GPT + UEFI (non CSM). Re-flash in **DD Image mode** if ISO mode fails. See [Troubleshooting](../troubleshooting.md) |
| Rufus wants to download extra files for the ISO | Some images need updated boot files; Rufus asks first | Allow it; it is part of making the stick bootable |

## Lessons

> 💡 **What I learned:** you can prepare the USB on the machine you're about to wipe; the installer runs from the flash drive. Download the right file (**Server**, **amd64**), not Desktop or ARM.

> 💡 **Also:** the two Rufus settings that matter, **GPT** and **UEFI (non CSM)**, are not Rufus trivia. They describe how the OptiPlex's firmware finds a bootloader, and the same two ideas come back in [step 9](09-bios-settings.md) (Secure Boot, AHCI) and [step 10](10-installing-ubuntu-server.md) (the `UEFI:` prefix on the boot menu entry).

## Glossary

| Term | Meaning |
|---|---|
| ISO | A single file holding an exact image of a disc: files, folders and boot sector. Named after the ISO 9660 CD standard |
| Bootable USB | A flash drive whose partition table and files let the firmware start an operating system from it |
| Flashing | Writing an image onto a drive so the drive becomes a copy of it, as opposed to copying a file onto it |
| Firmware / UEFI / BIOS | The program on the motherboard that runs at power-on and finds something to boot. UEFI is the modern form; "BIOS" is the old name people still use |
| CSM | Compatibility Support Module: UEFI's legacy-BIOS pretend mode. "UEFI (non CSM)" means not using it |
| MBR | Master Boot Record: the old partition table, 2TB limit, boot code in the first sector |
| GPT | GUID Partition Table: the modern partition table that UEFI expects |
| EFI System Partition | The small FAT32 partition on a GPT disk where UEFI looks for bootloaders |
| Secure Boot | UEFI feature that only runs signed bootloaders. Ubuntu's is signed, so it stays on |
| ISO Image mode | Rufus extracts the ISO's files onto a normal FAT32 partition |
| DD Image mode | Rufus copies the ISO byte for byte; a fallback when ISO mode will not boot |
| Hybrid image | An ISO built to boot whether written as files or as a raw copy; Ubuntu's are |
| amd64 / x86_64 | The 64-bit Intel and AMD PC architecture. The OptiPlex is this, not arm64 |
| LTS | Long-term support: an Ubuntu release maintained for five years |
| Portable (`.exe`) | A Windows program that runs without being installed; the `p` in `rufus-4.15p.exe` |
| Digital signature | A cryptographic stamp on a file proving who published it; Rufus is signed by Akeo Consulting |

---

[◀ Previous: Display, Keyboard and Network](../02-hardware/07-display-keyboard-and-network.md) · [🏠 Home](../../README.md) · [Next: BIOS Settings ▶](09-bios-settings.md)
