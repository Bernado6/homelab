# 10. Installing Ubuntu Server

Restart and tap **F12** for the boot menu.

My flash drive showed as **`UEFI: VendorCoProductCode 2.00`**, a generic name rather than the brand. The other entries were `Windows Boot Manager` and the `SAMSUNG …` SSD; don't pick those.

My first attempt **went back to Windows**. See [Troubleshooting](../troubleshooting.md) for what to check.

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

The install took about 15–20 minutes. If you open **View full log**, a line ending in `SUCCESS` is good; press **Close** to go back. At **Reboot Now**, remove the USB when asked, then log in at the text prompt. The password stays invisible as you type.

> 💡 **What I learned:** the two screens that really matter are **Storage** (correct disk, LVM off) and **SSH** (on). Everything else can be left at the defaults.

---

[◀ Previous: BIOS Settings](09-bios-settings.md) · [🏠 Home](../../README.md) · [Next: First Login and SSH ▶](11-first-login-and-ssh.md)
