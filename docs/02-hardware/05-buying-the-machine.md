# 5. Buying the Machine

This is the step where money leaves your account, so it is the step where a two-minute check is worth the most. The shortlist from [step 4](../01-planning/04-budget-and-software-choices.md) goes to sellers, offers come back in seller language, and the job is to translate that language into facts before paying.

## What

Message a local seller with the models and must-haves, compare what comes back, verify the exact unit, and buy. For me that was a used **Dell OptiPlex 7070 Micro**.

## Background: how to read a seller's offer

Sellers describe machines the way customers ask for them: brand, tier, generation, RAM, disk. That description leaves out most of what [step 1](../01-planning/01-goals-decide-specs.md) said matters. For each phrase in an offer, there is a follow-up question:

| Seller says | What it tells you | What you still need to ask |
|---|---|---|
| "Dell Core i7 8th gen" | Tier and generation | **Which i7?** An 8th-gen desktop i7 (like the i7-8700T) has 6 cores and 12 threads; the model string confirms it. |
| "Core i5 9th gen" | Tier and generation | **Which i5?** 9th-gen i5s have 6 cores; the T suffix tells you it is the low-power Micro variant. "ci5 9th gen" could be several chips. |
| "8GB RAM" | The amount | **One stick or two?** One 8GB stick leaves a free slot; two 4GB sticks do not ([step 4](../01-planning/04-budget-and-software-choices.md)). |
| "256GB SSD" | The size | **NVMe or SATA?** Both come in the M.2 shape that fits the slot. NVMe is what you want. |
| "Dell/Lenovo" | The seller has both in stock | **Which exact model?** OptiPlex 7060 or 7070? ThinkCentre M720q or M920q? |
| "Mini", "small", "desktop" | Not much | **Is it a Micro / Tiny / Mini**, the one-litre box, or a full-size desktop? Many sellers mostly sell laptops and use "desktop" loosely. |

### Comparing the two offers I got

I messaged a local seller with the models and specs I wanted. Two options came back:

| Option | Spec | Price |
|---|---|---|
| A | Dell Core i7 8th gen, 8GB RAM, 256GB SSD | KSh 38,000 |
| B | Dell/Lenovo Core i5 9th gen, 8GB RAM, 256GB SSD | KSh 35,000 |

Same RAM, same storage, KSh 3,000 apart. The difference is the CPU. An 8th-gen i7 would bring Hyper-Threading (12 threads on 6 cores); a 9th-gen i5 brings 6 cores without it. Step 1 asked for six cores, and both deliver six. For a homelab full of idle services, the extra threads were not worth the extra money, and the i5 is one generation newer. Option B it was.

## Background: what a service tag is, and what it tells you

Every Dell has a **service tag**: a short code (seven characters) on a sticker, usually on the underside or back of the case. Lenovo and HP have an equivalent serial number. Typed into the maker's support website, it returns:

| The support site shows | Why you care |
|---|---|
| The exact model | Settles 7060 versus 7070 without trusting the seller or the photo |
| The configuration it shipped with | CPU, RAM, disk as built. If the listing differs, parts were changed, which is worth knowing. |
| The original ship date | How old the machine actually is |
| Warranty status | Usually expired on a refurbished unit; still useful to know |
| Drivers, manuals and BIOS updates | You will want the BIOS page in [step 9](../03-ubuntu-server/09-bios-settings.md) |

Asking for the tag before paying costs nothing and tells you whether the seller knows what they are selling. Do not publish the tag anywhere afterwards; it identifies your specific machine (see the README on what stays out of this repo).

## Background: the parts that are easy to leave out of a deal

**The power adapter.** Micro PCs do not have a power supply inside; they run from an external brick, like a laptop. Dell's bricks use a specific barrel connector and the machine checks the adapter's identity, so a missing one is not a generic-charger problem, it is a part you must buy. Confirm it is in the box and in the price.

**Warranty.** A refurbisher's warranty is usually short, but it exists. "Used" from an individual usually has none. Ask, and get it in writing in the chat.

**RAM slots.** These machines have two SO-DIMM slots. The seller's answer to "how many slots are used?" changes the cost of your first upgrade:

| Inside the box | Upgrade to 16GB means |
|---|---|
| 1 × 8GB, one slot free | Buy **one** 8GB stick |
| 2 × 4GB, both slots full | Buy **two** 8GB sticks and discard the 4GB ones |

If the shop has spare sticks, this is the moment to ask for one to be fitted as part of the deal rather than buying it separately later.

## Checklist before paying

- [ ] Is it a **mini desktop**, not a laptop? (Many sellers mainly sell laptops.)
- [ ] **Real photos** of the actual unit, not a stock image
- [ ] **Exact CPU model** (e.g. i5-9500T, i7-8700T); "ci5 9th gen" isn't enough
- [ ] **Service tag** (sticker underneath), which you can check on Dell's support site
- [ ] **Power adapter** included
- [ ] **Warranty**
- [ ] Which **RAM slots** are used (1 of 2 makes upgrades cheaper)

### Why each item is on the list

| Item | What goes wrong without it |
|---|---|
| Mini desktop, not a laptop | A laptop has a built-in screen and battery you do not need, usually soldered or single-slot RAM, and poor cooling for running 24/7 |
| Real photos | A stock image can be of a 7060 while the unit is a 7070, or the reverse. Photos also show the condition, the ports, and whether the brick is in the shot. |
| Exact CPU model | The model string is the only way to know cores, threads and the T suffix. See the Background above. |
| Service tag | Verifies model and original configuration independently of the seller |
| Power adapter | A brick bought separately eats into the budget and may be a generic one the machine complains about |
| Warranty | Your only recourse if the SSD or fan fails in week two |
| RAM slots used | Decides whether the 16GB upgrade costs one stick or two |

## How: the conversation, in order

1. **Send the shortlist and must-haves, not the budget.** The model names from step 4, the generation floor, RAM, NVMe, Gigabit Ethernet, adapter included. Ask for specs and prices.
2. **Read the offers against the table above.** For each one, write the follow-up questions you still need answered.
3. **Ask the follow-ups.** Exact CPU model, real photos, service tag, how many RAM slots are used, is the adapter included, what warranty.
4. **Check the service tag** on the maker's support site. Confirm the model matches the photo and the CPU matches the answer.
5. **Negotiate the extras**, not just the price. A fitted RAM stick, a cable, a longer warranty. Shops often have spare parts and would rather include one than lose the sale.
6. **Pay, and keep the chat.** The specs, the price, the warranty terms and the service tag in one thread is your receipt.

**I bought:** Dell OptiPlex 7070 Micro (Option B).

## When

After step 4's shortlist exists. Before buying anything else: cables, RAM and adapters depend on which machine you end up with.

## Challenges and solutions

The purchase itself went smoothly. The traps are the ones the checklist exists for:

| Trap | What was going on | How to avoid it |
|---|---|---|
| OptiPlex 7060 and 7070 Micro look identical | Same case, same ports, one generation apart. A photo cannot tell them apart. | The exact CPU model string and the service tag can |
| "Dell/Lenovo" in one offer | The seller had both and would send whichever was on the shelf | Ask which model, and confirm the one in the photos is the one you pay for |
| "ci5 9th gen" as the whole CPU spec | Several different chips fit that description | Insist on the full model string before paying |
| Sellers who mainly stock laptops | "Desktop" gets used loosely and the mini PC may be a single unit they know little about | Ask the checklist questions; a seller who can answer them has handled the machine |
| Revealing the budget first | The offers arrive at exactly the budget | Ask for specs and prices before naming a number |
| Forgetting the brick | A machine that cannot be powered on until a second purchase arrives | Checklist item five |

## Lessons

> 💡 **What I learned:**
> - The **OptiPlex 7060 Micro** (8th gen) and **7070 Micro** (9th gen) look almost the same. Confirm the CPU, not just the photo.
> - Don't reveal your full budget in the first message. Ask for specs and prices first.
> - Ask the shop to include upgrades in the deal; they often have spare parts.

## Glossary

| Term | Meaning |
|---|---|
| Service tag | Dell's seven-character identifier for one machine; the key to its support page |
| Original configuration | What the machine shipped with, as recorded by the maker |
| Refurbished / used | Tested and resold with a short warranty / someone's old machine, usually as-is |
| Hyper-Threading | Intel's name for running two instruction streams on one core; why an i7 shows 12 threads on 6 cores |
| T suffix | Low-power (35W) version of a desktop CPU; what Micro PCs use |
| Power brick | The external power adapter a Micro PC runs from; model-specific connector |
| SO-DIMM slot | Where a laptop-sized RAM stick goes; these machines have two |
| M.2 | The slot shape both NVMe and SATA SSD sticks come in; ask which protocol |
| Stock image | A manufacturer's photo, not the unit being sold |
| Mini / Micro / Tiny | Dell's, Lenovo's and HP's names for the one-litre desktop form factor |

---

[◀ Previous: Budget, Specs and Software Choices](../01-planning/04-budget-and-software-choices.md) · [🏠 Home](../../README.md) · [Next: Checking What I Got ▶](06-checking-what-i-got.md)
