# 7. Display, Keyboard and Network

**Display:** I used a **Lenovo ThinkVision** monitor for the install, plus a USB keyboard. The OptiPlex 7070 Micro has DisplayPort outputs on the back, so check your monitor has a matching input or get the right cable or adapter.

After installing, the server runs **headless** (no monitor). I manage it from my laptop over SSH, and the monitor is only needed if something breaks.

**Network:** Windows showed **no Wi-Fi option**. That's not a fault: most office OptiPlex Micros ship without a Wi-Fi card. I used **wired Ethernet**, which is better for a server anyway.

If your router is far away:
- Long Ethernet cable (cheapest and most reliable)
- Powerline adapters
- A USB Wi-Fi dongle that explicitly supports Linux

> 💡 **What I learned:** plan where the server sits **next to the router** or within cable reach. A monitor and keyboard are only temporary needs; borrowing them is fine.

---

[◀ Previous: Checking What I Got](06-checking-what-i-got.md) · [🏠 Home](../../README.md) · [Next: Making the Ubuntu Install USB ▶](../03-ubuntu-server/08-making-the-install-usb.md)
