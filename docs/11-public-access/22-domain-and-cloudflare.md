# 22. Buying a Domain and Setting Up Cloudflare

## What

Buy a domain name, put its DNS on Cloudflare, and set up a Cloudflare account ready for the tunnel in [step 23](23-cloudflare-tunnel.md). This is the first money spent on the homelab since the machine itself, about USD 10 a year, and the first time anything in this project has a name on the public internet.

No cluster changes in this step. It's all decisions and a few web forms, but the decisions last for years, so they get a proper explanation.

## Background: what you are actually buying

### The pieces of a domain name

Take `portfolio.example.com`:

```text
portfolio . example . com .
   │           │       │   └─ the root (the trailing dot nobody types)
   │           │       └───── top-level domain (TLD), run by a REGISTRY (Verisign for .com)
   │           └───────────── second-level domain: the part you BUY and own the rights to
   └───────────────────────── subdomain: you create as many as you like, free, instantly
```

You don't buy `portfolio.example.com`. You buy `example.com`, and every name under it is yours to define.

### Registry, registrar, DNS host: three different jobs

This is the part most people never separate, and it's why some domain setups are confusing.

| Role | What they do | Who it is for `.com` | Who it is for you |
|---|---|---|---|
| **Registry** | Runs the TLD. Keeps the master list of who owns every `.com`. Sets the wholesale price. | Verisign | You never deal with them directly |
| **Registrar** | Sells you the name, bills you yearly, records your ownership with the registry. ICANN-accredited. | Cloudflare, Namecheap, Porkbun, GoDaddy, Truehost… | **Cloudflare**, in this step |
| **DNS host** (authoritative nameservers) | Answers the world's queries for your zone: "what's the A record for `portfolio.example.com`?" | Any DNS provider | **Cloudflare**, in this step |

The registrar and the DNS host **don't have to be the same company**. The link between them is the **NS (nameserver) records** the registrar publishes at the registry: "for `example.com`, ask these servers". Change those two lines and your DNS host changes. That's exactly what you'd do if you bought the domain elsewhere and still wanted Cloudflare's tunnel: register anywhere, point the nameservers at Cloudflare.

Buying at Cloudflare just means both jobs land in one place and the nameservers are already right.

### How your name reaches the rest of the internet

Step 21 showed the resolver chain from the client's side. Here it is from the domain owner's side:

```text
someone's resolver ──► root servers: "who handles .com?"       ──► Verisign's servers
                   ──► .com servers: "who handles example.com?" ──► NS records → Cloudflare's nameservers
                   ──► Cloudflare: "A record for portfolio.example.com?" ──► the answer you configured
```

Each hop is cached for its TTL. This is why changing nameservers at a registrar can take hours to be seen everywhere (the old NS answer is cached), while changing a record *inside* a zone that's already on Cloudflare takes seconds.

### Zone, records, apex

- A **zone** is everything under your domain, as managed by your DNS host.
- **Records** are the entries in it. For this project you'll mostly see `A` (name → IPv4), `CNAME` (name → name) and `TXT` (free text, used to prove you own the domain).
- The **apex** or root is the bare domain, `example.com` with nothing in front. Some things you can't do at the apex that you can on a subdomain (a CNAME, strictly speaking). Cloudflare quietly works around that with "CNAME flattening".

### WHOIS, privacy and the verification email

ICANN requires every domain to have a registered contact: name, address, email, phone. That record used to be public and was a spam magnet. Cloudflare **redacts it for free**; most registrars now do. You still have to enter real details, and ICANN requires you to **verify the email within 15 days** of registration or the domain is suspended. Watch for that email.

### Price: the number that matters is the renewal

Domains are sold like gym memberships. Some registrars advertise a first year at USD 1 and renew at USD 20. The honest comparison is **the renewal price**, every year, for as long as you want the name.

Cloudflare Registrar's policy is to charge **wholesale plus the ICANN fee, no markup**. For `.com` that is roughly USD 10 to 11 a year, and the same on renewal. That policy, more than any feature, is why it's the default choice here.

What that looked like for me: `benardkipngeno.com` cost **USD 10.46**, and the dashboard shows the renewal price as the same USD 10.46. A banner on the same page said Verisign, the `.com` registry, was raising its wholesale price to USD 11.17 from 1 November 2026, and that the new price would apply to renewals too. That's the registry setting the price and the registrar passing it through, which is the whole registry-versus-registrar distinction from the Background section, visible on one screen.

### Which TLD: deciding, not defaulting

The question you asked was "if `.com`, why?". Here's the reasoning, with the alternatives a portfolio owner actually considers:

| TLD | Yearly cost, roughly | For a professional portfolio | Catch |
|---|---|---|---|
| **`.com`** | USD 10 to 11 | **Universally recognised.** Recruiters and browsers assume it. Price is regulated and stable. | Good names are taken. Be ready with variants. |
| **`.dev`** | USD 12 to 14 | Signals "developer" clearly. Run by Google. Strong second choice. | The whole TLD is **HTTPS-only** (HSTS preloaded in browsers). Fine here, since Cloudflare gives you TLS, but plain HTTP will never work on it. |
| `.me` | USD 15 to 20 | Reads well for a personal site. | Montenegro's country code; slightly less "professional" in some eyes. |
| `.io` | USD 35 to 45 | Fashionable in tech. | Expensive, renewals creep up, and it's the country code of a territory whose sovereignty changed hands in 2024. Long-term status is uncertain. Avoid for something you want to keep. |
| `.ai` | USD 70 and up | Tempting for an ML engineer. | Very expensive, two-year minimum at many registrars. Not for a personal site. |
| `.co.ke` / `.ke` | KSh 1,000 to 1,500 | Local identity, cheap. | **Cloudflare Registrar doesn't sell it.** You'd buy at a KeNIC-accredited registrar (Truehost, HostPinnacle, Safaricom) and point the nameservers at Cloudflare. Works, one extra hop. |
| `.xyz`, `.site`, `.online` | USD 1 to 2 first year | Cheap. | Renewals jump to USD 10 to 15, and they have a spam reputation that some mail filters and corporate proxies act on. |

**Decision for a portfolio:** `.com` if a sensible form of your name is free, `.dev` if it isn't. `.co.ke` if local identity matters more to you than one less moving part. Everything else is either expensive, uncertain, or looks cheap.

**On the name itself.** It goes on a CV. `firstnamelastname.com` is the default. Variants if taken: `firstname-lastname`, initials plus surname, surname plus `.dev`. Avoid numbers, hyphens if you can, and anything you'd have to spell out loud.

## Background: what Cloudflare is

Cloudflare runs a network of servers in hundreds of cities that sits **in front of** other people's websites. A large share of the web passes through it. For a homelab the relevant products are all on the **free plan**:

| Product | What it does | Where it shows up in this guide |
|---|---|---|
| **Authoritative DNS** | Hosts your zone and answers the world's queries, fast, from everywhere | This step |
| **Registrar** | Sells domains at wholesale | This step |
| **Proxy ("orange cloud")** | Traffic to a record goes through Cloudflare's edge first: TLS termination, caching, DDoS absorption, hides your real address | Step 23, implicitly: tunnel hostnames are always proxied |
| **Zero Trust → Tunnels** | An outbound connection from your network to Cloudflare, so the edge can reach a service with no open ports | Step 23 |
| **Zero Trust → Access** | Put a login page from Cloudflare in front of anything on a tunnel | Later, if an admin UI ever needs to be public |

### Why Cloudflare for this project

| Reason | Detail |
|---|---|
| **The tunnel needs the zone there** | Step 23 only works if Cloudflare is the DNS host for the domain. That's the hard requirement. |
| **Free plan covers everything here** | DNS, proxy, TLS, tunnels, Access for up to 50 users. No trial, no card charge for these. |
| **Registrar at cost** | No markup, no renewal trap, WHOIS redaction included. |
| **One place** | Registrar, DNS and tunnel in one dashboard means one account to secure and no nameserver change to wait for. |
| **Public TLS for free** | Visitors get a certificate browsers already trust, issued and renewed by Cloudflare. No cert-manager work for the public name. |

### Why not the alternatives

| Alternative | Why not here |
|---|---|
| Namecheap, Porkbun, GoDaddy as registrar | Fine registrars (Porkbun especially), but you'd still move DNS to Cloudflare for the tunnel. One extra step and a nameserver propagation wait. Use them only if Cloudflare can't sell the TLD you want or can't take your card. |
| Route 53 (AWS) for DNS | Costs USD 0.50 a month per zone and doesn't give you a tunnel. Your old CloudFront setup lived here; no reason to stay. |
| A Kenyan registrar for `.co.ke` | The right choice if you want `.co.ke`. Point nameservers at Cloudflare afterwards and the rest of this guide is unchanged. |

### What Cloudflare will know

Honest trade-off: with the proxy and tunnel, Cloudflare sees every request to your public site in plain text at its edge, because that's where TLS terminates. For a public portfolio that's no concern. It's the reason admin UIs stay on `*.home.arpa` behind your own CA, and it's worth knowing before you put anything personal behind it.

## Why

- Step 23's tunnel can't exist without a domain on Cloudflare DNS.
- A portfolio on a name you own outlives any hosting choice. CloudFront, the homelab, or whatever comes next can all sit behind the same name.
- Learning: this is the owner's side of DNS. Step 21 was the client's side. After both you can follow any DNS problem end to end.

## When

Now, since nothing else is blocked on budget, and the tunnel is the next free step after it. Allow 20 minutes plus however long you spend choosing the name.

## How

### 1. Choose the name before opening any site

Write down three candidates in order, so you don't improvise at the checkout:

1. `firstnamelastname.com`
2. A variant: initials plus surname, or a hyphen
3. `surname.dev` or `firstnamelastname.dev`

### 2. Create the Cloudflare account

1. Go to `dash.cloudflare.com` and sign up with an email you'll keep for years. This account will own your domain.
2. Verify the email.
3. **Turn on two-factor authentication immediately**: profile → Authentication → Two-Factor Authentication. Whoever controls this account controls your name and everything pointing at it.

### 3. Register the domain

1. Left sidebar **Domains → Registrations**, then **Register domains**.
2. Search your first candidate. Cloudflare shows availability and the **yearly price**, which is also the renewal price.
3. If it's taken, try the next candidate. Don't settle for a weird spelling of your name; move to `.dev` instead.
4. **Add to cart → Checkout.** You'll be asked for:
   - **Registrant contact**: real name, address, phone, email. Required by ICANN. Cloudflare redacts it from public WHOIS at no cost.
   - **Payment**: card. If a Kenyan card is declined (it happens with some banks for USD web payments), try another card or PayPal. If nothing works, register at Porkbun or Namecheap and skip to the "bought elsewhere" note below.
   - **Auto-renew**: leave **on**. Losing a portfolio domain to a missed renewal is a classic.
5. Pay. The domain appears under **Websites** within a minute, already on Cloudflare's nameservers, zone active. No propagation wait, because registrar and DNS host are the same company.

### 4. The email you must not ignore

Within minutes you'll get an email titled something like *"Verify your email address for \<domain\>"* from Cloudflare on behalf of ICANN. **Click the link within 15 days** or the domain gets suspended. Do it now.

### 5. Lock it down

In **Domains → Registrations**, click **Manage** next to the domain:

| Setting | Value | Why |
|---|---|---|
| Auto-renew | On | See above. Cloudflare turns it on by default; the confirmation email says so. |
| Transfer lock | On | Stops anyone moving the domain to another registrar without unlocking here first. The registry locks every **new** domain for 60 days regardless, so the button shows as *Unlock* and greyed out at first. |
| **DNSSEC** | **Enable** (one click, Cloudflare handles the keys) | Cryptographically signs your zone so resolvers can detect forged answers. Free, and trivial when registrar and DNS host are the same place. Shows **Pending** for up to an hour while the record reaches the registry. |
| WHOIS privacy | On | Already on by default |

### 6. Look at the zone

Left sidebar **Domains → Overview**, click the domain, then **DNS → Records** in the domain's own menu. (Older guides and screenshots call this *Websites*; Cloudflare renamed it.) It's empty, apart from maybe a couple of defaults. Nothing points anywhere yet, and that's correct. Step 23 adds the first record automatically when the tunnel's public hostname is created.

Notice the **Proxy status** toggle on each record: orange cloud means "through Cloudflare's edge", grey means "plain DNS, hand out the address directly". Tunnel records are always orange.

### 7. Prepare Zero Trust

The tunnel lives in a separate part of the dashboard. Set it up now so step 23 starts at the tunnel itself:

1. Left sidebar, under **Protect & connect**, click **Zero Trust**. It opens a separate dashboard.
2. Pick a **team name**. It becomes `<team>.cloudflareaccess.com`, used only for Access login pages. Short and yours, it can't be changed easily.
3. Choose the **Free** plan. Cloudflare asks for a payment method even on the free plan; it isn't charged.
4. You land in the Zero Trust overview. Stop here.

### 8. Verify from the laptop

```bash
dig NS <your-domain> +short         # two Cloudflare nameservers, e.g. ada.ns.cloudflare.com
dig <your-domain> +short            # empty: no A record yet, which is correct
whois <your-domain> | grep -iE 'registrar:|creation|expir' | head -5
```

The NS answer proves the registry points at Cloudflare. The `whois` shows Cloudflare as registrar and the expiry a year out.

### If you bought the domain elsewhere

Everything above still applies, with one extra step. At Cloudflare: **Websites → Add a site → your domain → Free plan**. Cloudflare gives you two nameservers. At your registrar, replace the existing nameservers with those two. Then wait: the zone goes active when the registry's NS change is visible, usually within an hour, occasionally up to a day. The `dig NS` check above tells you when.

## Challenges and solutions

| Challenge | What was going on | Solution |
|---|---|---|
| Two-factor setup rejected the password | The field wants the Cloudflare account password, not the authenticator code or any other password. | Typed the right one. If you signed up with Google or Apple there is no password; use *Forgot password* to create one first. |
| Couldn't find "Websites" or "Domain Registration" in the dashboard | Cloudflare redesigned the sidebar. | *Websites* is now **Domains → Overview**; registration is **Domains → Registrations**; Zero Trust sits under **Protect & connect**. The How sections use the new names. |
| Transfer lock showed as greyed-out *Unlock* | The registry locks every new domain for 60 days. It's locked by rule, nothing to set. | Leave it. |
| DNSSEC showed *Pending* after enabling | The signing record takes up to an hour to reach the registry. | Wait. It goes active on its own. |
| "Related domains" box offering `.net`, `.org`, `.online` variants | Upsell. | Ignore. Variants of a personal name add cost and confusion, not protection. |

What went smoothly: the search, the card payment, and the zone being active the moment the purchase completed.

## Lessons

> 💡 **What I learned:**
> - **You buy the second-level name; everything under it is free.** Subdomains cost nothing and exist the moment you create a record.
> - **Registrar and DNS host are separate jobs**, joined by NS records at the registry. Knowing that is what makes "I bought it at X but host DNS at Y" make sense.
> - **Compare renewal prices, not first-year prices.** The cheapest way to buy a domain is from someone who charges the same every year.
> - **`.com` is the boring right answer for a professional name.** `.dev` is a good second. The fashionable TLDs are expensive, uncertain, or both.
> - **The account that owns the domain is the most important login you have.** Two-factor on, transfer lock on, auto-renew on, verification email clicked.
> - **DNS changes inside a zone are instant; nameserver changes are not.** It's caching all the way down, and buying and hosting in one place skips the slow part.

## Glossary

| Term | Meaning |
|---|---|
| TLD | Top-level domain: `.com`, `.dev`, `.ke` |
| Registry | The organisation that runs a TLD and owns the master list (Verisign for `.com`, KeNIC for `.ke`) |
| Registrar | The ICANN-accredited seller you buy from and renew with |
| DNS host / authoritative nameservers | The servers that answer queries for your zone |
| NS record | Published at the registry: "ask these nameservers about this domain" |
| Zone | Your domain and every record under it, as managed by the DNS host |
| Apex / root domain | The bare domain, `example.com`, with no subdomain |
| CNAME flattening | Cloudflare's trick for allowing CNAME-like behaviour at the apex |
| WHOIS | The public registration record; Cloudflare redacts your personal details |
| ICANN | The body that accredits registrars and requires the contact verification email |
| DNSSEC | Signatures on DNS answers so resolvers can detect tampering |
| Proxy status (orange / grey cloud) | Whether a record's traffic passes through Cloudflare's edge or goes direct |
| Zero Trust | Cloudflare's dashboard section for tunnels and Access |
| HSTS preload | A browser-side list of domains that must be HTTPS; the whole `.dev` TLD is on it |

---

[◀ Previous: Local DNS for the Whole House](../10-dns/21-local-dns-adguard.md) · [🏠 Home](../../README.md) · [Next: Public Access With Cloudflare Tunnel ▶](23-cloudflare-tunnel.md)
