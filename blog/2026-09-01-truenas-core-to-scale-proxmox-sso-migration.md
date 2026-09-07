---
layout: post.njk
title: "TrueNAS Core → Scale on Proxmox: A Real Migration, With SSO Wired In"
date: 2026-09-01
description: "The hands-on companion to the TrueNAS Core EOL news. A real Proxmox deployment walkthrough — migrating a Core pool to Scale as a VM, then wiring the new box into Authentik SSO so your family stops juggling NAS passwords. No SaaS listicle, just the commands and config that actually worked."
tags: ["truenas", "truenas-core", "truenas-scale", "proxmox", "zfs", "nas", "storage", "homelab", "self-hosted", "migration", "authentik", "sso", "freebsd", "freecore", "bsdnas"]
author: "Bryan Moon"
canonical: "https://devhandbook.io/blog/truenas-core-to-scale-proxmox-sso-migration"
---

# TrueNAS Core → Scale on Proxmox: A Real Migration, With SSO Wired In

Yesterday I wrote about [why TrueNAS Core is dead](/blog/2026-08-31-truenas-core-eol-migration-path/) and laid out the three paths forward: migrate to Scale, stay on FreeBSD with a fork, or leave TrueNAS entirely. That post was the "what happened and what are my options" piece.

This one is the "I actually did it, here's the exact config" piece.

Because here's the thing nobody tells you about the Core → Scale migration: **the ZFS part is easy. The identity part is where you bleed.** You can import a pool in five minutes. But then you've got a brand-new NAS with a brand-new set of users, and suddenly your family is back to "what's the password for the file server again?" — the exact problem you solved years ago.

So this post does two things at once. It walks through a real Core → Scale migration on Proxmox, and it wires the new Scale box into Authentik SSO so the migration isn't just a storage move — it's a chance to fix the auth mess for good.

This is the setup I run on my own hardware, adapted so you can follow along. No SaaS listicle, no "sign up for our sponsor." Just the commands and config that worked.

## The Setup I'm Migrating

Before I touch anything, here's the concrete starting point, because "migrate your NAS" is useless without a real topology:

- **Hypervisor:** Proxmox VE on a host I call `venus` (RFC1918 private subnet, redacted)
- **Old NAS:** TrueNAS Core running as a VM, with a single ZFS pool (`tank`) passed through via PCIe HBA passthrough
- **The pool:** `tank` with a few datasets — `tank/media`, `tank/backups`, `tank/documents` — and a handful of SMB shares
- **Identity:** Authentik running in Docker on a separate LXC, already protecting Proxmox, Grafana, and a few other services
- **The goal:** Get `tank` onto TrueNAS Scale, keep every byte, and make the new box's SMB shares authenticate through Authentik instead of a pile of local users

If your setup is different — bare-metal Core, no Proxmox, no SSO — the ZFS steps still apply. The SSO part is the bonus that makes the migration worth doing *properly* instead of just "good enough."

## Part 1: The ZFS Migration (The Easy Part)

### Step 0 — Export config, back up data

In the Core UI: **System → General → Save Config**. This gives you a `.tar` with users, shares, network settings, cron jobs, and everything that isn't your actual data.

Then back up your data. I'll say it again because it's the one step people skip: **a config export is not a data backup.** If `tank` holds anything irreplaceable, snapshot it and replicate it somewhere else before you start. ZFS migrations almost always go cleanly. "Almost always" is not a guarantee, and the one time you skip the backup is the one time the import fails.

### Step 1 — Build the Scale VM

Create a new VM in Proxmox. Don't reuse the old Core VM — build fresh, and keep the old one around until you've verified the new one. You want a rollback path.

```bash
# On the Proxmox host
qm create 201 \
  --name truenas-scale \
  --memory 16384 \
  --cores 4 \
  --cpu host \
  --net0 virtio,bridge=vmbr0 \
  --ostype l26 \
  --scsihw virtio-scsi-pci
```

Attach a boot disk (32 GB is plenty for the Scale OS) and, critically, **pass through the same HBA** that the old Core VM was using. If you used PCIe passthrough before, the IOMMU groups are already sorted — just point the new VM at the same device.

```bash
# Pass the HBA through (adjust the PCI address to yours)
qm set 201 --hostpci0 0000:01:00.0
```

Boot the Scale ISO, install to the boot disk, and let it come up. You now have a fresh Scale box with your drives visible but no pool imported yet.

### Step 2 — Import the pool

This is the part that makes people nervous, and it's the part that's actually trivial. ZFS is ZFS — the on-disk format is identical between FreeBSD and Linux. Your pool will import.

In the Scale UI: **Storage → Import Pool**, select `tank`, and confirm. Your datasets, snapshots, and data all come along.

If Scale warns you about a pool feature upgrade, **read it carefully.** A pool upgrade is one-way — once you upgrade, you can't go back to Core. If you're keeping the old Core VM as a rollback path, hold off on the upgrade until you're fully committed. You can import and use the pool without upgrading in most cases; the upgrade just enables newer ZFS features.

```bash
# Verify from the Scale shell
zpool status tank
zfs list -r tank
```

If `zpool status` shows no errors and your datasets are all there, the hard part is done. Your data survived the OS change.

### Step 3 — Import config, rebuild what doesn't transfer

Upload the config `.tar` you saved in Step 0 via **System → General → Upload Config**. Users, shares, and network settings come back. But two things won't:

1. **Jails.** Core used FreeBSD jails; Scale uses Docker. Any service you ran as a jail (Plex, Nextcloud, whatever) needs to be rebuilt as a Docker container or a TrueNAS App. This is the real time sink, and there's no shortcut — you're re-deploying those services.
2. **Some permissions.** FreeBSD and Linux map users differently. Expect to re-check share ACLs after import.

Run a scrub when you're done. It's cheap insurance that your data is actually intact, not just *present*.

```bash
zpool scrub tank
```

## Part 2: The SSO Tie-In (The Part Worth Doing)

Here's where most migration guides stop, and where I think they're wrong. You've just rebuilt your NAS. You have a fresh user database. This is the *perfect* moment to stop managing NAS users by hand and put the whole thing behind Authentik.

The goal: your family logs in once to Authentik, and their SMB shares, the TrueNAS web UI, and everything else just work — no per-service passwords, no sticky notes.

### What SSO can and can't do for a NAS

Let me be honest about the limits, because "SSO for your NAS" sounds magical and it isn't quite:

- **The TrueNAS web UI** supports OIDC natively. You can point it at Authentik and get real single sign-on for the admin interface. This is the clean win.
- **SMB shares** are the hard part. SMB doesn't speak OIDC. Your family's file access still goes through SMB authentication, which means local (or domain) users on the NAS itself. What SSO *can* do is centralize the *identity source* — Authentik can be backed by LDAP, and Scale can bind to that same LDAP directory, so there's one source of truth for who exists and what their password is.

So the realistic architecture is:

- **Authentik** is the identity provider and the single login your family sees
- **Scale binds to Authentik's LDAP outpost** for SMB user accounts, so you stop creating users in two places
- **The Scale web UI** uses OIDC against Authentik for admin login

That's the setup that actually removes password juggling, rather than pretending SMB can do OIDC.

### Step 1 — Enable Authentik's LDAP outpost

Authentik ships an LDAP provider that exposes your Authentik users as an LDAP directory. This is the bridge that makes SMB work.

In Authentik: **Directory → LDAP Providers → Create**, and configure:

- **Bind DN / password:** pick a service account (e.g. `cn=svc-truenas,ou=users,dc=example,dc=com`)
- **Search base:** your users OU
- **Bind mode:** direct binding

Then create an **Outpost** (type: LDAP) and assign the provider to it. Note the outpost's host and port — Scale will connect to it.

### Step 2 — Point Scale at the LDAP directory

In the Scale UI: **Credentials → Directory Services → LDAP**, and configure:

- **Hostname:** your Authentik LDAP outpost address
- **Base DN:** the search base you set in Authentik
- **Bind DN / password:** the service account from Step 1
- **Schema:** RFC2307 (or RFC2307bis if you use `memberOf` groups)

Once Scale can see the directory, your Authentik users become available as SMB users. Create the SMB shares and assign permissions to the LDAP users — no more hand-creating local accounts on the NAS.

### Step 3 — OIDC for the Scale web UI

Scale supports OIDC for admin login. In Authentik: **Applications → Create**, choose OIDC, and set:

- **Redirect URI:** `https://truenas.example.com/ui/sessions/signin` (adjust to your Scale URL)
- **Scopes:** `openid profile email`

Then in Scale: **System → General → OIDC**, and fill in the provider URL, client ID, and client secret from Authentik.

Now your Scale admin login redirects to Authentik. You log in once, and you're in — with MFA if you've enabled it (you should).

### Step 4 — Put it behind your reverse proxy

If you're already running a reverse proxy (I use Cloudflare Tunnels for external access), route the Scale web UI through it and let Authentik's forward-auth outpost protect it. This is the same pattern as any other homelab service — the point is that your NAS is now just another SSO-protected app, not a special snowflake with its own login.

## What I'd Do Differently Next Time

The migration itself was smooth. The parts that cost me time were all *around* the migration, not the migration:

1. **I should have inventoried my jails first.** I had a Plex jail and a small Nextcloud jail, and I didn't realize how much config lived in them until I had to rebuild them as Docker. Write down every jail, what it does, and where its data lives *before* you wipe Core.
2. **The LDAP schema matters.** I spent an hour chasing a "user not found" error that turned out to be a schema mismatch between Authentik's LDAP outpost and Scale's default. RFC2307bis vs RFC2307 — check it early.
3. **Don't upgrade the pool until you're sure.** I held off on the ZFS feature upgrade for a week while I verified everything, and I'm glad I did. It gave me a clean rollback to the old Core VM if anything had gone sideways.

## The Bottom Line

The Core → Scale migration is not the scary part. The ZFS import is genuinely easy — your data is never actually trapped, because ZFS is ZFS regardless of the OS underneath it.

The *valuable* part is treating the migration as an excuse to fix the things you've been putting off. For me, that was identity. I'd been running a pile of local NAS users for years, and the migration forced me to centralize them in Authentik and wire the whole thing into SSO. Now my family has one login, the NAS is just another protected app, and I'm not the password-reset helpdesk anymore.

If you're still on Core, the clock is ticking — not because your data is in danger tomorrow, but because every month you wait is a month of unpatched risk accumulating. Back up your config, back up your data, and pick a path. And when you do the migration, don't just move the storage — use the moment to fix the auth mess too.

---

*Migrating off Core? Hit a wall with the LDAP or OIDC config? I'd genuinely like to hear how it went — find me on [GitHub](https://github.com/bryanmoon19) or drop a note in the comments.*

*Related reading: [TrueNAS Core Is Dead — Your Migration Path](/blog/2026-08-31-truenas-core-eol-migration-path/), [Self-Hosted NAS on Proxmox](/blog/2026-08-07-proxmox-nas-truenas-anas-turnkey/), and the [Self-Hosted Auth/SSO Showdown](/blog/2026-08-08-self-hosted-auth-sso-showdown/).*
