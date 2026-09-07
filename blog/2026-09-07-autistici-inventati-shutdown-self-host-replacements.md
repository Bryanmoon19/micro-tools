---
layout: post.njk
title: "A Privacy Collective Ran for 20 Years. The US Designated It a Terrorist Org. Here's Your Exit Plan."
date: 2026-09-07
description: "Autistici/Inventati — the Italian privacy collective that hosted email, VPN, XMPP, and web hosting for two decades — was designated a Specially Designated Global Terrorist organization and shut down overnight. If you relied on it (or anything like it), here's how to run each replacement on your own box."
tags: ["autistici", "inventati", "privacy", "self-hosted", "email", "vpn", "xmpp", "jabber", "web-hosting", "homelab", "docker", "de-google", "own-your-stack", "censorship", "infrastructure"]
author: "Bryan Moon"
canonical: "https://devhandbook.io/blog/2026-09-07-autistici-inventati-shutdown-self-host-replacements"
affiliate: true
cta: true
---

On August 26, 2026, the US Treasury designated **Autistici/Inventati (A/I)** — an Italian collective that had run privacy infrastructure since 2001 — as a Specially Designated Global Terrorist organization. Within days, its main `.org` domain went dark. The collective announced it was shutting down entirely.

Let that land for a second. A group that provided **email, VPN, XMPP chat, web hosting, and mailing lists** to activists, journalists, and privacy-conscious users for a quarter-century was designated a terrorist organization — and the infrastructure vanished overnight. Not because of a hack. Not because of a funding shortfall. Because a government put it on a list.

The HN thread hit 552 points. The story is everywhere. But here's what nobody is writing: **the practical part.** What did A/I actually provide, and how do you run each replacement yourself so the same thing can't happen to you?

This post is that guide. It's not a news recap — it's a wake-up call and an exit plan.

## What Actually Happened

The timeline, briefly:

- **Aug 26** — US Treasury sanctions the Italian hosting provider, designating A/I as a Specially Designated Global Terrorist (SDGT) organization. The stated basis: A/I allegedly hosted infrastructure for groups the US considers terrorist organizations.
- **Aug 28** — A/I's main `.org` domain goes dark. The collective's services — email, VPN, XMPP, web hosting, mailing lists — begin to fail.
- **Sept 6** — The collective confirms it is shutting down. Two decades of infrastructure, gone.

The legal and political questions are real and worth debating. But for the thousands of people who *used* A/I, the immediate question is simpler: **my email, my VPN, my chat, my website — what do I do now?**

The answer, for anyone with a homelab or a $5 VPS, is: **run it yourself.** That's the entire thesis of this site, and this is the clearest possible demonstration of why it matters. A service you don't control can be taken from you by a government, a company, or a single bad actor. A service you run on your own box can't.

## What A/I Provided (and What to Run Instead)

A/I was a full privacy stack. Here's each piece, mapped to a self-hosted replacement you can deploy today.

### 1. Email → Mox or Stalwart

A/I's email was its flagship service — free, privacy-respecting mailboxes for people who didn't want Gmail reading their mail.

**The replacement:** I wrote a full guide to this in August — [Self-Hosted Email in 2026: The Stack That Actually Delivers to Gmail](/blog/2026-08-16-self-hosted-email-2026-stack-that-delivers-to-gmail/). The short version: the old Postfix+Dovecot stack is a maintenance tax, and the modern single-binary servers (Mox, Stalwart, WildDuck, Maddy) handle DKIM/SPF/DMARC out of the box.

The honest caveat from that post still applies: **deliverability is the hard part.** Gmail, Outlook, and Yahoo silently drop mail from unknown IPs. If you self-host email, you need a clean IP, proper DNS records, and realistic expectations. But for a personal mailbox that talks to other self-hosters and privacy-conscious contacts, it works.

### 2. VPN → WireGuard (or Headscale/Tailscale)

A/I ran a VPN service for its users. This is the *easiest* thing to replace — VPN is the most mature self-hosted category there is.

**The replacement:** [WireGuard + Pi-hole: The Complete Privacy Stack](/blog/2026-04-21-wireguard-pihole-privacy-stack/) — one Docker Compose file, ten minutes, and you have an encrypted tunnel plus network-wide ad blocking. If you want a mesh instead of a point-to-point tunnel, Headscale (the self-hosted Tailscale control server) is the move.

The key difference from A/I's VPN: **you're the only user.** No shared IP with strangers, no "who else is on this exit node" question. Your traffic is yours.

### 3. XMPP/Jabber chat → Prosody or ejabberd

A/I ran XMPP servers — the federated, open chat protocol that predates and outlives every walled-garden messenger.

**The replacement:** Prosody (lightweight, Lua, single config file) or ejabberd (heavier, more features). Both are mature, both run in Docker, and both speak the same XMPP protocol A/I's users already know. Your existing XMPP contacts and clients (Conversations on Android, Dino on desktop) keep working — you just point them at your own server.

The honest note: XMPP's federation means you're not *isolated* by self-hosting — you can still chat with people on other servers. That's the whole point of the protocol, and it's why A/I's XMPP users have a clean migration path.

### 4. Web hosting → Caddy + a VPS

A/I hosted websites for its users. This is the piece that's genuinely hard to replace *for free* — you need a server with a public IP.

**The replacement:** A $5 VPS running Caddy (automatic HTTPS, single binary, reverse proxy in one line). Point a domain at it, and Caddy handles TLS certs and routing. If you want to keep it on your homelab instead, a tunnel (Cloudflare Tunnel, or the self-hosted alternatives I covered in [Cloudflare Tunnel Alternatives](/blog/2026-09-03-self-hosted-tunnel-alternatives-gopher-frp-rathole/)) gets you a public URL without opening ports.

The honest cost: **web hosting is the one thing you can't fully self-host for free.** You need either a VPS or a tunnel. But $5/month buys you a server nobody can designate away.

### 5. Mailing lists → Listmonk

A/I ran mailing lists for activist and community groups.

**The replacement:** Listmonk — a self-hosted newsletter and mailing-list manager. Single binary, Postgres backend, clean UI, and it handles the subscribe/unsubscribe/bounce dance that makes mailing lists actually work. It's the closest thing to a drop-in replacement for what A/I's list users lost.

## The Deeper Lesson

Here's the thing that should stick with you, beyond the specific tools.

A/I wasn't a startup that ran out of money. It wasn't a service that got acquired and enshittified. It was a **collective that ran for 20 years** — longer than most companies — and it was killed by a *designation*. A list. A government action that didn't require a trial, a warrant, or even a specific accusation that held up in court.

The lesson isn't "don't trust privacy collectives." A/I did real, valuable work for two decades. The lesson is **don't make any single third party the sole custodian of your infrastructure** — no matter how trustworthy they are. Because the failure mode isn't always "they sell out." Sometimes it's "they get designated, and everything they host goes dark in 48 hours."

This is the same argument I've been making across this site — [de-Googling your Android](/blog/2026-09-02-degoogling-android-grapheneos-self-hosted-stack/), [self-hosting your location history](/blog/2026-08-23-google-location-history-self-host/), [running your own Nitter](/blog/2026-09-06-self-hosted-twitter-nitter-xcancel/). The A/I shutdown is just the most dramatic proof yet: **if you don't run it, you don't own it.**

## The 80/20 Exit Plan

If you were an A/I user and you want to be un-killable by next week, here's the priority order:

1. **Email first** — it's the hardest to migrate (you have to update every account that emails you) and the most painful to lose. Start the Mox/Stalwart migration now, even if it takes a week to get deliverability right.
2. **VPN second** — WireGuard is a ten-minute deploy. Do it tonight.
3. **XMPP third** — Prosody is a twenty-minute deploy, and your existing contacts keep working.
4. **Web hosting and lists last** — these need a VPS, so they're a "this weekend" project, not a "tonight" project.

The total cost: a $5 VPS and a few hours. The total benefit: infrastructure that no designation, no acquisition, and no policy change can take from you.

---

*This post is about the practical migration path, not the politics of the designation. If you want the full legal and political context, the primary sources are the [US Treasury press release](https://home.treasury.gov/news/press-releases/sb0616) and the [State Department designation](https://www.state.gov/releases/office-of-the-spokesperson/2026/08/designation-of).*
