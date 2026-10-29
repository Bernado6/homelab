# 7. Display, Keyboard and Network

A server spends its life with no screen attached, which makes it easy to forget that it needs one on exactly one day. This page is about that day's kit, the network port that is the server's real interface, and the question from [step 0](../01-planning/00-before-you-start.md) that decides both: where the box lives.

## What

Gather what the install needs (a monitor with the right input, a keyboard, an Ethernet cable to the router), understand why the machine has no Wi-Fi and why that is fine, and pick the spot where it will sit for years.

## Background: why a server needs a screen exactly once

Everything after the install happens over **SSH**: a text session from the laptop to the server over the network. But SSH only exists once there is an operating system running an SSH server and holding a network address. Before that point, nothing can reach the machine remotely, and the only things that can show you anything are the firmware and the installer, both of which draw to a locally attached display:

```text
 power on
   │
   ▼
 BIOS / UEFI  ── needs the screen (settings in step 9, boot menu in step 10)
   │
   ▼
 Ubuntu installer ── needs the screen and keyboard (step 10)
   │
   ▼
 Ubuntu Server boots, OpenSSH is running, the router gave it an IP
   │
   ▼
 ssh from the laptop (step 11)  ── unplug the monitor; it is headless from here
```

A machine run with no display, keyboard or mouse is **headless**. After step 11 the only reasons to plug a screen back in are the ones where the network path itself is broken: a boot failure, a wrong network setting, a forgotten password. That is rare, and it is why borrowing a monitor is a perfectly good plan.

## Background: DisplayPort vs HDMI

Both carry digital video and audio over one cable. The difference that matters on install day is **which one your machine has and which one your monitor has**.

| | DisplayPort (DP) | HDMI |
|---|---|---|
| Where you find it | Business PCs, including the OptiPlex Micro's back panel; many office monitors | TVs, consumer monitors, laptops |
| Connector | Rectangular with one corner cut off; often a latch | The familiar thin trapezoid |
| Smaller variant | Mini DisplayPort | Mini and micro HDMI (the Raspberry Pi uses micro) |

The combinations:

| PC output → monitor input | Works with |
|---|---|
| DP → DP | A DisplayPort cable |
| DP → HDMI | A **DisplayPort-to-HDMI** cable or adapter. Works on practically every PC, because the DP port can speak HDMI in that direction. |
| DP → VGA (older monitors) | An **active** DP-to-VGA adapter, because VGA is analogue |
| HDMI → DP | Needs an active converter and is not worth it; find a different monitor |

So: look at the back of the machine, look at the back of the monitor, and buy or borrow the one cable that joins them. Then, on install day, make sure the monitor is switched to that input; "No signal" is more often the wrong input than a bad cable.

## Background: keyboards, mice and the installer

The firmware and the Ubuntu Server installer both understand ordinary **USB keyboards** without any driver. A wireless keyboard with its own USB receiver dongle also works, because the dongle presents itself as a plain keyboard. A **Bluetooth** keyboard does not work before the operating system is up, because pairing is an OS feature. The Ubuntu Server installer is **text-based**: arrow keys, Tab, Space and Enter. No mouse is needed and none was used.

## Background: wired versus wireless, for a server

[Step 1](../01-planning/01-goals-decide-specs.md) made the general case. For this machine there are three specific reasons:

1. **There is no Wi-Fi card.** Most office OptiPlex Micros ship without one. Windows showing no Wi-Fi option is not a fault; the slot is empty.
2. **The address has to stay put.** Your router runs **DHCP**: when a device joins, the router lends it an IP address from a pool for a limited time (a **lease**). Later, in [step 14](../05-networking/14-dhcp-reservation.md), the router is told to always hand this machine the same address, and it recognises the machine by the **MAC address** of its network port. The wired port's MAC is fixed and always present; it is the natural identity for a server.
3. **Headless and wireless is a bad pair.** If a Wi-Fi password changes or the signal drops, a headless machine silently vanishes and the only fix is a monitor and a keyboard. A cable does not do that.

Ethernet cable runs can be long: the standard allows up to 100 metres, so distance is almost never the problem. Routing the cable neatly is.

## Background: where the box lives

A Micro PC is quiet and cool, but it still breathes. What the spot needs:

| Need | Why | What to avoid |
|---|---|---|
| Airflow around the vents | The fan pulls air in one side and pushes it out the other | A closed cabinet, a drawer, pressed against a wall, a pile of things on top |
| A power socket, ideally one the UPS can later sit on | It is on 24/7 | Extension leads shared with a kettle or iron |
| Ethernet reach to the router | Everything above | Planning on Wi-Fi "for now" |
| Away from dust and heat | Dust clogs the heatsink; heat makes the fan work harder | The floor, a windowsill in the sun, next to a cooker |
| Somewhere a monitor can be plugged in for an hour | Install day, and the rare rescue | A spot so tight you have to unplug it to get at the ports |

## How

### 1. Display and keyboard

**Display:** I used a **Lenovo ThinkVision** monitor for the install, plus a USB keyboard. The OptiPlex 7070 Micro has DisplayPort outputs on the back, so check your monitor has a matching input or get the right cable or adapter.

After installing, the server runs **headless** (no monitor). I manage it from my laptop over SSH, and the monitor is only needed if something breaks.

Kit checklist for install day:

- [ ] Monitor, and you know which inputs it has
- [ ] The one cable or adapter that joins the machine's DisplayPort to the monitor's input
- [ ] USB keyboard (wired, or wireless with its own dongle)
- [ ] The USB flash drive from [step 8](../03-ubuntu-server/08-making-the-install-usb.md)
- [ ] The power brick
- [ ] Ethernet cable, plugged in **before** you boot the installer, so it can get an address from the router and confirm the internet works (the installer's Mirror screen in [step 10](../03-ubuntu-server/10-installing-ubuntu-server.md) is where you will see that it did)

### 2. Network

**Network:** Windows showed **no Wi-Fi option**. That's not a fault: most office OptiPlex Micros ship without a Wi-Fi card. I used **wired Ethernet**, which is better for a server anyway.

Plug one end into the Ethernet port on the back of the machine and the other into any LAN port on the router (not the WAN or internet port, which is for the fibre connection). The port lights on both ends should come on. That is the whole network setup for the install; the router's DHCP does the rest.

### 3. If your router is far away

- Long Ethernet cable (cheapest and most reliable)
- Powerline adapters
- A USB Wi-Fi dongle that explicitly supports Linux

In more detail:

| Option | How it works | Reliability | Watch out for |
|---|---|---|---|
| Long Ethernet cable | A cable, run along skirting or under a carpet | Best. Full Gigabit, no interference. | Cable clips and a bit of patience |
| Powerline adapters | A pair of plugs that send network data over the house's electrical wiring | Usually fine, slower than a cable | Both plugs must be on the same electrical circuit; extension leads and surge protectors degrade it |
| USB Wi-Fi dongle | A Wi-Fi radio on a USB stick | Weakest, and a headless machine on Wi-Fi is fragile | It must have a driver in the Linux kernel. Many cheap dongles only ship Windows drivers; look for "Linux" in the listing and a known chipset |

If you end up on option two or three, treat it as temporary and still plan the cable.

### 4. Pick the spot

Walk the house with the table from the Background section. My machine ended up wired directly to the router, which removed the whole question. If yours cannot, the cable is still the answer; it is just a longer one.

## When

Before [step 8](../03-ubuntu-server/08-making-the-install-usb.md). You will make the install USB on this machine while Windows is still on it, which also needs the screen and keyboard, so have the kit together first.

## Challenges and solutions

Nothing failed on install day for me. These are the problems this page exists to prevent:

| Trap | What is going on | Solution |
|---|---|---|
| "No signal" on the monitor | The monitor is on a different input, or the cable is DP-to-HDMI plugged the wrong way round | Cycle the monitor's input button; make sure the DP end is in the PC |
| An HDMI-only monitor and a DP-only machine | They do not plug into each other | One DisplayPort-to-HDMI cable or adapter |
| No Wi-Fi in Windows, assumed to be broken | There is no Wi-Fi card. Normal for an office Micro. | Use the cable |
| USB Wi-Fi dongle not recognised by Ubuntu | No Linux driver for its chipset | Buy one that states Linux support, or run a cable |
| Powerline adapters slow or dropping | Different circuits, or plugged through a surge protector | Same circuit, straight into the wall |
| Cable "almost" reaches | A stretched cable gets unplugged by a vacuum cleaner | Buy the next length up; Ethernet does not care about an extra few metres |
| Server in a closed cabinet | Fan runs constantly and the CPU throttles | Give it air on both sides |

## Lessons

> 💡 **What I learned:** plan where the server sits **next to the router** or within cable reach. A monitor and keyboard are only temporary needs; borrowing them is fine.

## Glossary

| Term | Meaning |
|---|---|
| Headless | Running with no monitor, keyboard or mouse; managed over the network |
| SSH | Secure Shell: the encrypted terminal session used to manage the server after install |
| DisplayPort (DP) | The digital video connector on business PCs; the OptiPlex Micro has these |
| HDMI | The digital video connector on TVs and most consumer monitors |
| Active adapter | A converter with electronics inside, needed when the signal must change form (DP to VGA, HDMI to DP) |
| VGA | The old analogue video connector with 15 pins; needs an active adapter from DP |
| USB HID | The standard class a keyboard or mouse presents as; why USB keyboards work in the BIOS without drivers |
| Text-based installer | An installer driven by keyboard only; Ubuntu Server's is one |
| Wi-Fi card | A small M.2 module; absent in most office Micros |
| DHCP | The router service that lends IP addresses to devices for a limited time |
| Lease | The period a DHCP address is valid for before it is renewed |
| MAC address | The fixed hardware address of a network port; how the router recognises the server in step 14 |
| LAN port / WAN port | The router's ports for your devices / the single port for the internet connection |
| Powerline | Networking over the house's electrical wiring using a pair of plug-in adapters |
| Linux driver / chipset | The kernel code that supports a specific piece of hardware; a dongle without one does not work |

---

[◀ Previous: Checking What I Got](06-checking-what-i-got.md) · [🏠 Home](../../README.md) · [Next: Making the Ubuntu Install USB ▶](../03-ubuntu-server/08-making-the-install-usb.md)
