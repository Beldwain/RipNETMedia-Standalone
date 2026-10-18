# Changelog

All notable changes to RipNETMedia Standalone are listed here.
This project uses [calendar-aware semantic versioning](https://semver.org).

## [1.0.0] — 2026-10-XX

First public release of the single-PC Windows edition.

### Added
- **Setup wizard** — a plain-language, step-by-step first run: disc drive check,
  ripping tools, where your media goes (auto-detects mapped drives and network
  folders, pick Movies/TV/Music, test access), how to rip, and optional metadata
  keys. Re-runnable any time from Settings.
- **Audio CDs** → FLAC or MP3. MP3 is produced from a FLAC rip for reliable,
  fully playable files, with correct tags.
- **DVD / Blu-ray / UHD** ripping via MakeMKV, optional HandBrake re-encoding.
- **Automatic metadata** from TMDB, TheTVDB, OMDb, MusicBrainz and Discogs
  (all optional; music is identified with no key).
- **NAS / local folder** organising with **Plex**, **Jellyfin/Emby** or **generic**
  naming (your choice).
- **Duplicate detection** before copying, with **Copy Anyway**, **Replace**, or
  **Save as a different edition** (Director's Cut, Live, Remastered, …).
- **Retry** and **Cancel** on any job; jobs interrupted by a restart are recovered.
- **About** and **Acknowledgements & Licensing** pages; a dismissible first-run
  disclaimer.
- **Export / Import settings** so you can back up and restore your configuration.
- **Download diagnostics** — a shareable bug-report bundle with secrets removed.
- Browser **dashboard** (local only) and a **tray app** with "Start with Windows".

### Notes
- Free for personal use; business use requires a license; redistribution is not
  permitted. See `LICENSE.txt`.
- Not yet code-signed — Windows SmartScreen may prompt on first launch
  (More info → Run anyway).
- DVD/Blu-ray ripping needs the free MakeMKV, installed separately; its beta key is
  refreshed periodically at makemkv.com.

[1.0.0]: ../../releases/tag/v1.0.0
