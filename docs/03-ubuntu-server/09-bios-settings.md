# 9. BIOS Settings

Restart and tap **F2** at the Dell logo.

| Setting | Value | Why |
|---|---|---|
| **SATA Operation** | **AHCI** (if it says RAID On) | Otherwise the Ubuntu installer may not see the SSD |
| **AC Recovery** | **Power On** | The server turns itself back on after a power cut |
| **Secure Boot** | Left **ON** | Ubuntu supports it |

> 💡 **What I learned:** **AC Recovery → Power On** is a must if you get power cuts. Without it, the server stays off after an outage.

---

[◀ Previous: Making the Ubuntu Install USB](08-making-the-install-usb.md) · [🏠 Home](../../README.md) · [Next: Installing Ubuntu Server ▶](10-installing-ubuntu-server.md)
