# 8. Making the Ubuntu Install USB

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

## Steps

1. Download the Ubuntu Server ISO to **Downloads** on the OptiPlex, *not* onto the flash drive.
2. Download `rufus-4.15p.exe`.
   - Chrome blocked it as an "uncommon download". I kept it via `Ctrl + J` → **Keep**.
   - To check it's genuine: right-click → **Properties → Digital Signatures**, which should show **Akeo Consulting**.
3. Open Rufus with the flash drive plugged in:
   - **Device:** the flash drive (check the size!)
   - **Boot selection:** the Ubuntu ISO
   - **Partition scheme:** GPT
   - **Target system:** UEFI (non CSM)
4. **START** → *Write in ISO Image mode* → wait for **READY**.

> 💡 **What I learned:** you can prepare the USB on the machine you're about to wipe; the installer runs from the flash drive. Download the right file (**Server**, **amd64**), not Desktop or ARM.

---

[◀ Previous: Display, Keyboard and Network](../02-hardware/07-display-keyboard-and-network.md) · [🏠 Home](../../README.md) · [Next: BIOS Settings ▶](09-bios-settings.md)
