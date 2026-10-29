# 9. BIOS Settings

Three settings, five minutes, and two of them decide whether the next step works at all. The firmware is the one layer of the machine that neither Ubuntu nor Kubernetes can fix for you, so it is worth understanding what each switch does before flipping it.

## What

Open the OptiPlex's firmware setup, make sure the SSD is presented in a way the Ubuntu installer understands (**AHCI**), tell the machine to switch itself back on after a power cut (**AC Recovery → Power On**), and leave **Secure Boot** as it is.

Restart and tap **F2** at the Dell logo.

| Setting | Value | Why |
|---|---|---|
| **SATA Operation** | **AHCI** (if it says RAID On) | Otherwise the Ubuntu installer may not see the SSD |
| **AC Recovery** | **Power On** | The server turns itself back on after a power cut |
| **Secure Boot** | Left **ON** | Ubuntu supports it |

## Background: what the BIOS is, and F2 vs F12

The **firmware** is a small program stored on a chip on the motherboard. It runs the instant the machine gets power, before any disk is read: it checks the RAM, finds the drives, decides which one to boot from, and hands control to a bootloader. On a modern PC the firmware is **UEFI**, but everyone still calls it "the BIOS", including Dell's own screens.

Firmware settings are kept in a small battery-backed memory on the board, which is why they survive a reinstall of the operating system and why wiping Windows does not reset them. Whatever you set here stays set until you change it again.

Dell gives you two keys at the logo, and they do different things:

| Key | Opens | Use it for |
|---|---|---|
| **F2** | **BIOS Setup**: the full settings program | Changing settings permanently. This step. |
| **F12** | **One-Time Boot Menu**: a list of bootable devices for this boot only | Starting the installer from the USB stick once, without changing the permanent boot order. [Step 10](10-installing-ubuntu-server.md). |

Tap the key repeatedly from the moment the screen lights up; if Windows starts loading you were too late, restart and try again. On this generation of Dell firmware the setup screen is graphical, works with the mouse as well as the keyboard, and groups settings into categories down the left-hand side. The three settings below live under **System Configuration**, **Power Management** and **Secure Boot**.

## Background: AHCI vs RAID On, and why the installer cares

A drive is only useful if the operating system has a **driver** that speaks the same language as the controller the drive sits behind. Dell's **SATA Operation** setting chooses that language for the storage controller, and it applies to the NVMe slot as well as to SATA ports despite the name.

| Mode | What it is | Who understands it |
|---|---|---|
| **AHCI** | Advanced Host Controller Interface: the open, standard way to talk to drives. An NVMe SSD in this mode is simply visible as an NVMe device. | Every operating system, every installer, every live USB |
| **RAID On** | The controller is handled by Intel **Rapid Storage Technology (RST)**, Intel's own layer that can combine drives into RAID sets and do caching tricks. In this mode drives are hidden behind the RST controller. | Windows, with Intel's RST driver. Linux installers often see **no disk at all** |
| Disabled | The controller is off | Nothing; do not use |

Dell ships many OptiPlexes with **RAID On** because Windows installs with the RST driver and some Windows features assume it. A single-SSD machine gains nothing from RAID mode; there is nothing to RAID. For Linux it is purely an obstacle: you reach the Storage screen in the installer and the list of disks is empty.

```text
 RAID On                                  AHCI
 ───────                                  ────
 Ubuntu installer                         Ubuntu installer
   │ "any disks?"                           │ "any disks?"
   ▼                                        ▼
 Intel RST controller  ✗ no Linux driver  Standard AHCI/NVMe controller  ✓
   │                                        │
   └── SSD (invisible)                      └── SSD: SAMSUNG … 256GB
```

Why not keep RAID On and install Intel's driver into the installer? Because the fix is one menu change, and because AHCI is what Ubuntu is built and tested for. The comparison was not close:

| Option | Verdict |
|---|---|
| **Switch to AHCI** ✅ | One setting. The installer sees the SSD as a plain NVMe device. Nothing to maintain later. |
| Stay on RAID On | Windows keeps booting (irrelevant: it is about to be erased), Linux installer may show no disk. Some newer firmware can be coaxed with kernel parameters; not worth it. |

One side effect to know: the Windows that is currently installed was set up expecting RAID On, so after the switch Windows may fail to start. That does not matter here. The install USB from [step 8](08-making-the-install-usb.md) is already made, and Windows is about to be overwritten. It is also why the order of these steps is what it is: make the USB first, change the firmware second.

## Background: AC Recovery, and why a server needs it

A desktop PC that loses mains power stays off when power returns, because that is what a person at a desk expects. A server is the opposite: nobody is there to press the button, and everything running on it (and in [step 21](../10-dns/21-local-dns-adguard.md), DNS for the whole house) is down until someone notices.

**AC Recovery** is the firmware setting that decides what happens when mains power comes back:

| Option | After power returns, the machine… | Good for |
|---|---|---|
| Power Off (the default) | Stays off until the button is pressed | Desktops |
| **Power On** ✅ | Switches itself on every time | Servers. What I set. |
| Last Power State | Comes back on only if it was on when power was lost | A compromise: a machine you had deliberately shut down stays down |

**Power On** is the simplest promise: if there is electricity, the server is running. The cost is that if you shut it down on purpose and the power then flickers, it comes back; for a machine that is meant to be on 24/7 that is the behaviour you want. In Nairobi, where cuts are a normal part of the week, this single setting is the difference between a homelab that recovers on its own and one that needs a walk to the router cupboard.

It is not a replacement for a **UPS** (uninterruptible power supply, a battery that bridges the gap and lets the machine shut down cleanly). A UPS is on the roadmap. AC Recovery is the free half of the plan and it goes in on day one.

## Background: Secure Boot, and why Ubuntu is fine with it

**Secure Boot** is a UEFI feature. With it on, the firmware will only run a bootloader whose digital signature matches a key stored in the firmware. Its purpose is to stop malware that installs itself *before* the operating system, where no antivirus can see it.

The question for a Linux install is whether the Linux bootloader is signed by a key the firmware trusts. For Ubuntu the answer is yes:

```text
 firmware (Secure Boot on)
   │  trusts: Microsoft UEFI CA key (present in every PC's firmware)
   ▼
 shim  ── Ubuntu's first-stage bootloader, signed by Microsoft
   │  trusts: Canonical's key
   ▼
 GRUB  ── signed by Canonical
   ▼
 Linux kernel  ── signed by Canonical
```

Each link checks the next, and the first one is signed with the same Microsoft key that Windows uses, so an unmodified Ubuntu boots on a Secure Boot machine with no changes. That is why the setting is **left on**, and why turning it off is the *last* item in the step 10 troubleshooting list rather than the first: it is almost never the reason a USB stick does not boot.

| Option | Verdict |
|---|---|
| **Leave Secure Boot on** ✅ | Ubuntu boots, the machine keeps its boot-path protection, nothing to remember later |
| Turn it off | Only helps with unsigned bootloaders or third-party kernel modules that are not signed. Not needed for Ubuntu Server or K3s. |

One consequence you may meet later: with Secure Boot on, any **out-of-tree kernel module** (some Wi-Fi dongle drivers, proprietary GPU drivers) has to be signed before the kernel will load it. On a wired server with no GPU work this never comes up.

## When

After [step 8](08-making-the-install-usb.md), because the SATA change can stop Windows booting and you want the USB finished first. Immediately before [step 10](10-installing-ubuntu-server.md): the same restart can take you from F2 to F12.

## How

### 1. Open BIOS Setup

Restart the machine. From the moment the Dell logo appears, tap **F2** once a second until the setup screen opens. If Windows begins loading instead, let it, restart, and tap earlier.

### 2. SATA Operation → AHCI

Under **System Configuration → SATA Operation**, look at the current value. If it says **RAID On**, choose **AHCI**. If it already says AHCI, there is nothing to do. The firmware may show a warning that changing the mode can prevent the installed operating system from booting; accept it, because that operating system is Windows and it is about to go.

Expected result: on the Storage screen in step 10, the installer lists the **SAMSUNG … 256GB** SSD.

### 3. AC Recovery → Power On

Under **Power Management → AC Recovery**, choose **Power On**. There is nothing to verify now; the proof arrives at the next power cut, when the machine boots itself and `ssh` works again without anyone touching it.

### 4. Secure Boot: look, do not change

Under **Secure Boot**, confirm that **Secure Boot Enable** is on and leave it. Nothing else on this screen needs touching for Ubuntu.

### 5. Save and exit

Choose **Apply** (or **Save**) and then **Exit**. The machine restarts. On that restart, go straight to [step 10](10-installing-ubuntu-server.md) and tap **F12** instead.

Nothing else in the firmware needs changing for Ubuntu Server or K3s. Leave the boot sequence, memory and CPU settings at their defaults.

## Challenges and solutions

Nothing went wrong for me on this step. These are the traps it prevents, and they show up in the *next* step if missed:

| Trap | What is going on | Solution |
|---|---|---|
| Installer's Storage screen shows no disks | SATA Operation is still **RAID On**; the installer has no Intel RST driver | Come back here: **System Configuration → SATA Operation → AHCI**. See [Troubleshooting](../troubleshooting.md) |
| Windows will not start after the AHCI change | Windows was installed for RAID mode | Expected and harmless; it is being replaced. Boot the USB instead. |
| Server stays off after a power cut | AC Recovery is at its **Power Off** default | **Power Management → AC Recovery → Power On** |
| USB stick refuses to boot and advice says "disable Secure Boot" | Almost always a different cause: wrong flash mode, wrong port, wrong entry | Work through the step 10 list first. Secure Boot off is the last resort, not the first. |
| Missed the F2 window and Windows loaded | Keys are only read during the logo | Restart and start tapping as soon as the screen lights up |
| Changed a setting and it did not stick | Exited without **Apply** | Apply, then Exit |

## Lessons

> 💡 **What I learned:** **AC Recovery → Power On** is a must if you get power cuts. Without it, the server stays off after an outage.

> 💡 **Also:** every one of these three settings is about *who can see what*. AHCI lets the installer see the disk, AC Recovery lets the machine see that power is back, and Secure Boot lets the firmware see that the bootloader is genuine. None of them is Ubuntu-specific knowledge; they apply to any PC you turn into a server.

## Glossary

| Term | Meaning |
|---|---|
| BIOS / UEFI / firmware | The program on the motherboard that runs before the OS. UEFI is the modern form; "BIOS" is the name everyone still uses |
| F2 / F12 (Dell) | BIOS Setup (change settings permanently) / One-Time Boot Menu (pick a device for this boot only) |
| SATA Operation | Dell's name for the storage controller mode; applies to the NVMe slot too |
| AHCI | The standard, open controller interface that every OS understands |
| RAID On / Intel RST | Intel's Rapid Storage Technology mode; hides drives behind a controller that Linux installers often cannot see |
| Driver | The OS code that speaks to a particular piece of hardware; no driver, no disk |
| AC Recovery | What the machine does when mains power returns: Power Off, Power On, or Last Power State |
| UPS | Uninterruptible power supply: a battery that bridges short cuts and allows a clean shutdown. On the roadmap. |
| Secure Boot | UEFI only runs signed bootloaders. Ubuntu's `shim` is signed by Microsoft, so it works |
| shim / GRUB | Ubuntu's first-stage bootloader (Microsoft-signed) and the main bootloader it hands over to |
| Out-of-tree kernel module | A driver not shipped with the kernel; must be signed to load under Secure Boot |
| CMOS / NVRAM | The small battery-backed memory that keeps firmware settings across reinstalls |

---

[◀ Previous: Making the Ubuntu Install USB](08-making-the-install-usb.md) · [🏠 Home](../../README.md) · [Next: Installing Ubuntu Server ▶](10-installing-ubuntu-server.md)
