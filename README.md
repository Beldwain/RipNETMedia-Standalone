# RipNETMedia Standalone

A simple, one-PC Windows app that rips your own **DVDs, Blu-rays and audio CDs**
to your computer or NAS — with automatic titles, cover art, and folder names that
Plex, Jellyfin and Kodi understand. No separate server, no accounts, no monthly fee.

> **Free for personal use.** Business use requires a licence, and redistribution is
> not permitted. See [LICENSE](LICENSE.txt).

This repository hosts the **downloads and documentation**. The source code is private.

Why is this a "standalone" version? There is a server with multi-client version that is 
being worked on right now that will be license based.  This version does not offer as much
as the server based version but we believe it offers enough to make people happy to
start using it. They both have similar development vectors but this one is ready now, 
the server version still has validations and other testing to complete before we are 
comfortable releasing it for purchase.

![Step-by-Step Easy setup](https://github.com/user-attachments/assets/64c012bd-8e1d-4a45-b9dc-b7774efe2cf8)
---

## Download

1. Go to the [**Releases**](../../releases/latest) page and download
   `RipNetMedia-Standalone-Setup.exe`.
2. (Optional) Verify it against the published `SHA256SUMS.txt`.
3. Run the installer and follow the setup wizard.

> **Heads-up — Windows SmartScreen.** The app isn't code-signed yet, so Windows may
> show *"Windows protected your PC."* Click **More info → Run anyway**. This is
> expected; see the [Installing guide](../../wiki/Standalone‐Installing) for pictures.

**You need:** 64-bit Windows 10 or 11. For DVDs/Blu-rays, the free
[MakeMKV](https://www.makemkv.com) (installed separately). **Audio CDs need nothing
extra** — the CD ripper is built in.

## What it does

- Rips **audio CDs** to **FLAC** (exact copy) or **MP3** (plays everywhere).
- Rips **DVD, Blu-ray and UHD** via MakeMKV, with optional HandBrake re-encoding.
- Looks up **titles, episodes and cover art** automatically (TMDB, TheTVDB, OMDb,
  MusicBrainz, Discogs — all optional and free; music works with no sign-up).
- Organises and **copies to your NAS or a local folder**, with names tuned for
  **Plex, Jellyfin/Emby or a generic layout**.
- **Spots duplicates** before copying and lets you keep a **different edition**
  (Director's Cut, Live, Remastered, …).
- A friendly **setup wizard**, a browser **dashboard**, a **tray app**, one-click
  **settings backup/restore**, and a **diagnostics** bundle for bug reports.

## Documentation

- [Installing](../../wiki/Standalone‐Installing)
- [First-time setup](../../wiki/Standalone‐First‐Time‐Setup)
- [Ripping your first disc](../../wiki/Standalone‐Ripping‐Your‐First‐Disc)
- [Settings & backups](../../wiki/Standalone‐Settings‐and‐Backups)
- [Troubleshooting](../../wiki/Standalone‐Troubleshooting)
- [FAQ](../../wiki/Standalone‐FAQ)

## Reporting bugs

Please open an [Issue](../../issues) and attach the **diagnostics** file
(in the app: **About → Download diagnostics**). It contains logs and system info
with your API keys removed.

# RipNETMedia — Lessons Learned from ARM

This is not a new idea, we took the idea of  Automatic Ripping Machine (ARM) project
(https://github.com/automatic-ripping-machine/automatic-ripping-machine) and the
specific design decisions RipNETMedia makes is to create a user friendly version that 
is easier to get started and available without running extra hosting or VM's.

ARM is a Python/Flask-based headless ripping system. It has been in active development
for years and represents a real-world body of lessons that we have learned from.
We are building fresh (no shared code) but using ARM as a reference for what to get right.

The ARM developers have dedicated an extraordinary amount of time, skill, and passion to building
something that genuinely works for thousands of people across countless hardware configurations.
Their contributions to the open-source community have enriched the home media space in ways that
are difficult to overstate. RipNETMedia does not exist to criticize that work — it exists because
of it. We are respectfully and genuinely in awe of what the ARM team has built, and we encourage 
everyone using RipNETMedia to support the ARM project and its contributors for everything they 
have given to this community.

## Legal

RipNETMedia is for making **personal backup copies of discs you own**. Copyright and
anti-circumvention laws vary by country; you are responsible for using it lawfully
where you live. RipNETMedia does not decrypt copy-protected discs itself and is not
affiliated with MakeMKV, HandBrake, Plex, Jellyfin, or any disc publisher.

Includes third-party software under their own licences (cyanrip — LGPL-2.1,
FFmpeg — LGPL-3.0, and others); see the in-app **Acknowledgements & Licensing** page
and `THIRD-PARTY-NOTICES.txt`.

© 2026 Whateverco.com. All rights reserved.

<!-- TODO before publishing: confirm the repo path used in the ../../ links above
     (they assume this is the RipNETMedia-Standalone repo with the wiki enabled). -->
