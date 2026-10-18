# RipNetMedia — Windows Standalone

A single-machine edition of RipNetMedia for Windows: one installer, no separate
server, no login. It reuses the same web dashboard as the full client-server
build, so a user who later outgrows one PC keeps the UI they already know.

Status: working end to end (video via MakeMKV, audio CDs to FLAC via cyanrip, MP3 via FLAC + FFmpeg).
Version: `VERSION` (single source -- see "Version" below).
User guide: [`wiki/Windows-Standalone.md`](../wiki/Windows-Standalone.md).

## Architecture

```
 browser  ──HTTP──►  ripnetmedia-service.exe  (Windows Service, LocalSystem, auto-start)
 (dashboard)          │  127.0.0.1:47899 only  (loopback = the security boundary; no auth)
                      ├─ REST API + serves the built web dashboard (web/ → server/web → <install>\web)
                      ├─ JobEngine: disc → identify → metadata → rip → organize → transcode → verify → copy
                      ├─ SQLite  %ProgramData%\RipNetMedia\ripnetmedia.sqlite   (logs in .\logs)
                      └─ shells out to: makemkvcon (video), cyanrip (audio CDs), ffmpeg (FLAC->MP3), HandBrakeCLI (optional)

 ripnetmedia-standalone-tray.exe  ── thin: tray icon, balloon notifications, "Open Dashboard",
                                     "Rescan Drives", autostart. All real UI is the web dashboard.
```

| Path | What |
|---|---|
| `service/src/JobEngine.*` | The pipeline / state machine. Reuses `client/src/drive`, `ripper`, `organizer`, `nas`. |
| `service/src/api/RestApi.*` | Loopback REST API + static dashboard + SPA fallback. Route list is documented in `RestApi.h`. |
| `service/src/ToolLocator.*` | Finds/validates MakeMKV, HandBrake, cyanrip, FFmpeg; cleans user-typed paths. |
| `service/src/MbDiscId.h` | MusicBrainz disc-ID from a CD table of contents (header-only, unit-tested). |
| `service/src/Db.*`, `Settings.*` | SQLite wrapper; settings reader. `Db` also adds settings introduced after v1 on startup. |
| `tray/src/` | Tray app + `TrayApi` (REST client used by the tray and the headless tests). |
| `db/schema.sqlite.sql` | Schema + seed (applied only to an empty DB). |
| `installer/` | NSIS script + `THIRD-PARTY-NOTICES.txt`. |
| `tools/cleandb.cpp` | One-off maintenance tool (see "Maintenance"). |
| `../client/src/ripper/CyanRipRipper.*` | Audio CD ripper wrapper (shared location beside `MakeMkvRipper`). |
| `../client/src/ripper/FlacToMp3Converter.*` | FLAC -> MP3 with FFmpeg (libmp3lame), with per-track verification. |
| `service/src/NasCopy.h` | Folder copy with size verification + `normalizeNasPath()` (shared with the tray). |

### One frontend for both editions
`web/` is built once (`web/vite.config.ts` → `server/web`). At load, `AuthContext`
calls `/health`; the standalone service includes a `"service":"ripnetmedia-service"`
field the full server does not. When present the app skips login, hides
server-only pages (Clients, Users, Roles, Notifications, Workflows, Maker Portal,
Data), and uses standalone variants (Setup wizard steps, NAS path "Folder path").
Standalone has no WebSocket, so pages poll every 3 s.

The standalone API mirrors the server's JSON shapes on purpose (jobs use `id`,
settings carry `setting_key/setting_value`, etc.) so the same pages work. Keep it
that way when adding routes.

## Tools

| Tool | Needed for | Where it comes from |
|---|---|---|
| **cyanrip** | Audio CDs → FLAC | **Bundled** in the installer (`<install>\tools\cyanrip-win64.exe`, LGPL-2.1). Currently the **nightly 2026-08-21 (f8ebf4)** build. Only ever asked for FLAC now (its MP3s were unreliable). |
| **FFmpeg** | FLAC → MP3 (MP3 profiles) | **Bundled** (`<install>\tools\ffmpeg.exe` + `ffmpeg-LICENSE.txt`; **LGPL-3.0** build (`--enable-version3`) with libmp3lame; staged from `dist\tools\`). Setting `tools.ffmpeg_path`, else bundled copy, else PATH -- any candidate whose `-encoders` list lacks libmp3lame is skipped (checked once per file, cached). Optional, but `GET /api/tools` marks it **required** (with a warning if missing) while an MP3 profile is the default audio profile. |
| **MakeMKV** | DVD / Blu-ray / UHD | User installs (proprietary; cannot be bundled). |
| **HandBrake CLI** | Optional transcode | User installs. |

`ToolLocator` resolves each tool **per job** (no service restart needed): the
saved setting if it points at a real file, else auto-detect (Program Files on every
fixed drive, then PATH), else bundled (cyanrip). Saving a `tools.*_path` setting
cleans it first: quotes/spaces stripped, folder → exe inside it, missing `.exe`
added, MakeMKV's GUI `makemkv.exe` → `makemkvcon64.exe`. `GET /api/tools` reports
status for the wizard's **Ripping Tools** step and the Settings panel.

## Audio CDs
1. Identify: read the TOC (`IOCTL_CDROM_READ_TOC`) **before** the volume label —
   Windows mounts audio CDs as a CDFS volume named "Audio CD". Compute the
   MusicBrainz disc ID → fingerprint `cd-<discid>`, then the normal MusicBrainz lookup.
2. Rip: `CyanRipRipper` runs `cyanrip -I` first (to pick among multiple matching
   releases, preferring the release our lookup matched), then rips to
   `<rip_dir>\job_N\<Artist>\<Album>\N - Title.flac` + `.log` + `.cue`.
   Progress is emitted as MakeMKV-style `PRGV:` lines so `JobEngine` is unchanged.
   **Always FLAC**: cyanrip's own MP3 output was never reliably playable.
3. Format comes from the job's quality profile (`Audio-FLAC` / `Audio-MP3`).
   Profile choice: most specific match wins, then the `is_default` flag, then id.
   For **MP3** the job goes to *transcoding* ("Converting FLAC to MP3 (320 kbps) with FFmpeg",
   `encoding_progress`) and `FlacToMp3Converter` converts every FLAC next to itself:
   `ffmpeg -hide_banner -nostdin -y -i in.flac -map 0:a:0 -map_metadata 0 -c:a libmp3lame -b:a <kbps>k -compression_level 2 -id3v2_version 3 -write_id3v1 1 -f mp3 out.mp3.part`
   (CBR at `audio.mp3_bitrate_kbps`, clamped 32-320; tags copied to ID3v2.3 + ID3v1; file names --
   track numbers and multi-disc `1.01 - ` prefixes -- unchanged). Each MP3 must be at least half
   the size its duration x bitrate predicts (min 4 KB) and decode with no errors
   (`ffmpeg -v error -i x.mp3 -f null -`). Only when **every** track passes are the FLACs deleted and
   the `.cue`'s `FILE "x.flac" WAVE` lines rewritten to `FILE "x.mp3" MP3` (the `.log` is kept as is).
   On any failure the job fails ("MP3 conversion failed: ... -- the FLAC files are still in ...").
   Verified on real hardware (Iggy Pop, *Lust for Life*: 9 tracks, 320 kbps CBR, all decode cleanly).
4. Organize/transcode are skipped for audio; the Artist folder is copied to the
   NAS path whose media type is **Audio** (set this on the NAS Paths page) -- see
   "Copying to the NAS" below for how the copy happens.
5. Any 0-byte audio file fails the job (in CyanRipRipper and again in the engine's verify step,
   which also requires at least one file of the profile's format).
6. **Multi-disc albums:** disc 2 shares disc 1's title and year, so the title+year duplicate
   guard and the NAS "title already exists" check are skipped for audio CDs (the exact-disc
   fingerprint check still applies). Tracks use cyanrip's multi-disc naming (`1.01 - Title`,
   `2.01 - Title`, per-disc log/cue) so the discs share one Artist\Album folder without colliding.
   Albums ripped before this change have unprefixed names (`01 - Title`) for disc 1.

Settings (added on startup for existing DBs): `tools.cyanrip_path`, `tools.ffmpeg_path`,
`audio.drive_offset_samples` (default 0; only affects AccurateRip bit-exactness),
`audio.mp3_bitrate_kbps` (default 320).

## Copying to the NAS (tray does the copy)
The service runs as `LocalSystem` in Session 0. Windows creates mapped drive letters
(`N:`, `S:`, ...) and stores NAS logons **per logon session**, so the service can never
see them - the server-edition client avoided this by running as an ordinary program in the
user's session. Standalone therefore splits the work:

1. The engine rips/organizes into `paths.rip_dir\job_N` as before. If the job's NAS path
   is visible to the service (e.g. a UNC share it can authenticate to) it copies directly.
2. Otherwise it stores a row in `pending_transfers`, leaves the job in **transferring**
   ("Waiting for the RipNetMedia tray app to copy...") and frees the drive.
3. `TransferWorker` in the tray (user session, where the drive letters exist) polls
   `GET /api/transfers/pending`, copies with `NasCopy.h` (verifies every file's size),
   reports `POST /api/transfers/{id}/progress` and `.../complete`.
4. On success the service deletes the local working copy and marks the job **done**; on
   failure the job is **failed** with the reason and the files stay in the working folder.
   Cancelling a waiting job removes it from the queue (the tray stops copying).

Consequences: copies only happen while the tray is running in the logged-in user's session.
The tray starts itself with Windows (see "Tray autostart" below), so this normally just works;
a job waits in *transferring* until the tray is up and survives service restarts.
`pending_transfers` is created by `Db` on startup, so existing databases upgrade automatically.

### Tray autostart (self-managed, #10)
The **installer no longer writes the HKCU `...\Run` key.** Under an elevated installer `HKCU`
is the *installing* account's hive -- an admin installing for a standard user would write
autostart into the wrong profile. Instead the **tray** (which always runs in the real user's
session) manages its own autostart in `EnsureAutoStartDefault()` (`tray/src/ripnetmedia-tray.cpp`),
run once per launch:
- A separate preference value `HKCU\Software\RipNetMedia\Tray\AutoStartChoice` = `on`/`off`
  distinguishes **never set** (first run) from a **deliberate off**. This is *not* the Run key;
  it is the remembered choice.
- **Never set** -> enable autostart for this user and record `on`. **On** -> re-assert the Run
  key (idempotent; self-heals a moved/reinstalled exe path). **Off** -> leave it absent and never
  silently re-enable.
- The **Start with Windows** menu toggle is authoritative: it flips the Run key *and* records the
  choice, so an explicit OFF persists.
- HKCU-only, no admin, never touches the service -- it cannot fight either.

Limitation: the uninstaller runs elevated, so its `DeleteRegValue HKCU ...\Run` only clears the
*uninstalling* account's hive, not necessarily the user who ran the tray. The tray removes its own
Run key when the user toggles Start with Windows off; a leftover Run entry after uninstall is
harmless (it points at a deleted exe, so Windows just skips it).

### "Already on the NAS" check (tray and service)
The service cannot scan a NAS it cannot see, so for mapped drives the **tray** checks before it
copies. Both sides call the same `findExistingMedia(dir, discPrefix, stem, scope)` (`NasCopy.h`) with
arguments the service computes (`JobEngine::copyCheckFor`) and serves in
`GET /api/transfers/pending` as `check_path`, `check_disc_prefix`, `check_stem`, `check_scope`, `on_conflict`:

| Content | Checked (`scope`) | Conflict when |
|---|---|---|
| Movie, Plex/generic | `NAS\Title (Year)`, top level (`movie`) | a media file, **except** `Extras\` and other `{edition-...}` files |
| Movie, Jellyfin | `NAS\Title (Year)`, top level (`flat`, `stem`) | a media file with the same name (any extension) -- other versions are not duplicates |
| Movie extras disc | `NAS\Title (Year)\Extras` (`flat`) | any media file |
| Full-disc backup | `NAS\Title (Year) [DISC]` (`tree`) | any media file (BDMV\STREAM\…) |
| TV | `NAS\Show\Season N` (or `\Extras`) (`flat`) | any media file |
| Audio | `NAS\Artist\Album`, top level (`flat`) | a media file; multi-disc rips (`2.01 - ...`) only files of the same disc number |

**The check never looks above this folder.** When the folder cannot be told apart -- the working
copy is gone, or holds several album/season folders -- there is **no check at all** (`check_path`
empty), never a wider, destructive scope (QA D1). A missing or empty folder, or one with only
artwork/cue/log files or only `Extras\`, is not a duplicate (QA D4).

On a conflict with `on_conflict: review` the tray does not copy and sends
`POST /api/transfers/{id}/conflict {existing_path, file_count, total_bytes, samples[], files[]}`; the job
goes to `duplicate_review` with `duplicate_reason = nas_exists`. `files[]` (each `{path, size}`) is **every
file the user is shown** and the only thing a later Replace may delete -- stored in `jobs.duplicate_nas_files`
(the samples / count / size are also kept for the UI). The job keeps its working files and its
`pending_transfers` row (the plan a decision copies), and the drive stays free (restart recovery does not
re-lock it; reinserting the disc does not spawn a clutter job). The tray's balloon says "Already on your
NAS" (TrayState re-announces a job that comes back to review). An older tray build ignores the new fields
and copies as before.

Where the service CAN see the NAS it runs the same check before every copy it makes itself
(`serviceNasCheck`: normal pipeline, Retry copy, Duplicate Review decisions), with the same result.
It also checks movies/TV **before ripping** when the default folder template lets it predict the
destination (exact folder name, media files required -- it used to match any folder whose name merely
started with the title, e.g. "Alien" matched "Aliens (1986)", and empty folders counted). Audio is
only checked after the rip (folder names come from cyanrip).

**Replace deletes only the reviewed files.** `replaceReviewedMedia()` deletes only the files stored in
`duplicate_nas_files` that are still on disk with the same size -- nothing is searched again, so a file
added after the review, another album, artwork or a folder can never be deleted (QA D2/D3). It refuses
**before deleting anything** when the ripped source is gone, when the checked folder is not strictly inside
the destination (`isStrictlyInside`), or when a listed file is outside it; and re-asks (back to review) if
the destination is no longer the one that was reviewed. Both the tray and the service go through it.

### Duplicate Review decisions
`POST /api/jobs/{id}/resolve-duplicate` (`JobEngine::resolveDuplicate(jobId, DuplicateResolution)`):

| Decision | Before the rip (library match, or the service saw it on the NAS) | After the rip (`nas_exists` + verified files in the working folder) |
|---|---|---|
| Proceed (`add_new`) | read the disc again with `force_rip`; copy into the existing folder (`nas_conflict_action = merge`) | re-queue the copy with `on_conflict: merge` (no check) |
| Replace | as above, `nas_conflict_action = replace` (+ `replace_library_id`) | re-queue with `on_conflict: replace`; the copier deletes only the reviewed files (`replaceReviewedMedia`), then copies |
| Edition | `jobs.edition` set; applied after organizing | working copy renamed for the edition (`renameForEdition`, audio also re-tagged -- see below), copy re-queued to the new destination, which is **checked again** |
| Cancel | cancelled | cancelled; files kept in the working folder (Retry copies them, checked again) |

The copy after a decision goes through the service if it can reach the NAS, else the tray. The
`duplicate_nas_*` fields are cleared when the job leaves review for good (Proceed, Edition, Cancel) and when
it finishes (done/failed); Replace keeps them until the copy, because they are the list it may delete.
Pipeline Retry clears them and `nas_conflict_action` (the edition stays). Edition names are sanitised
(`sanitizeEdition`: drops control chars, `< > : " / \ | ? *` and `{}[]()`, collapses spaces, trims, max **60
characters** counted as code points): 400 if nothing is left, 409 if the library already has that edition of
the same disc (audio) or title + year (video). `GET /api/jobs/{id}/edition-preview?edition=` returns the
resulting folder/file/destination (from the working copy after the rip, from the metadata before it).
`jobs.edition` / `edition_target` and `library.edition` are added by `Db` on startup; the library
`filename` of an edition is the edition's own album folder (audio) or main file (Jellyfin movie), not the
shared artist/movie folder. Shown on Job detail and Library.

### Editions and media-server naming
Setting `naming.media_server` = `generic` (default) | `plex` | `jellyfin` (added on startup; saving
normalises case/spaces, maps `emby` to `jellyfin`, rejects anything else with 400; Settings shows a
dropdown). Only names **with an edition** depend on it (`EditionNaming.h`); base naming never changes.

| | generic (Kodi / none) | plex | jellyfin (also Emby) |
|---|---|---|---|
| Movie folder | `Title (Year) - X` | `Title (Year) {edition-X}` | `Title (Year)` (shared) |
| Movie file | `Title (Year) - X.mkv` | `Title (Year) {edition-X}.mkv` | `Title (Year) - X.mkv` |
| Movie extras | in its own folder | in its own folder | `Extras\name (X).mkv` (so versions' extras do not overwrite each other) |
| Full-disc backup | `Title (Year) - X [DISC]` | `Title (Year) {edition-X} [DISC]` | `Title (Year) - X [DISC]` |
| Album | `Album (X)` | `Album (X)` | `Album (X)` |
| TV show | `Show (Year) [X]` | `Show (Year) [X]` | `Show (Year) [X]` |

Why: Plex reads `{edition-...}` in the movie folder/file name as an edition of the same movie. Jellyfin and
Emby group files named `Folder name - label.ext` in ONE movie folder as versions of that movie. Kodi has no
edition naming; a separate `Title (Year) - X` folder scrapes as the same movie and keeps the files apart --
and is harmless in every other server, so it is the default (a user who never opens the setting gets names
that work everywhere; `{edition-...}` would be noise outside Plex, and Jellyfin's shared folder would mix
files for Kodi users). Neither Plex nor Jellyfin has an edition convention for **TV** or **music**, so one
convention is used for all: a bracketed suffix on the show folder (the server still matches the show by
name -- it may merge both into one show, showing duplicate episodes; use a separate library to keep them
apart) and `Album (X)`. **Audio editions are re-tagged** (`AlbumRetagger`, FFmpeg `-c copy` so audio and
artwork are untouched): the ALBUM tag becomes `Album (X)` and the release / release-group / disc
MusicBrainz IDs are removed (both the FLAC Vorbis and ID3 spellings), so a music server that groups by tags
lists the edition separately instead of merging it. A re-tag failure keeps the renamed folder and logs a
warning. These conventions were not verified against a running media server here.

**Drive-letter NAS paths:** `path("S:") / "Metallica"` is the *drive-relative* `S:Metallica` (jobs
showed "Job complete: S:Metallica"). `normalizeNasPath()` (NasCopy.h) turns `X:` into `X:\` (and
`X:name` into `X:\name`) when NAS paths are saved via the API, wherever the engine builds a
destination / scans the NAS / checks reachability, when the pending-transfer list is served, and
again in the tray's `TransferWorker` (rows queued by older builds). Existing rows are not rewritten.

NAS path media types decide which path a job uses: set Movies = Movie, TV = TV Series,
Music = **Audio**. Paths with no type are only a catch-all when no typed path matches.

## Job lifecycle rules
- **No ripping before setup is complete.** Until `setup.completed` is true the engine ignores every
  disc insertion: the startup scan is skipped, drive-monitor events are ignored (logged once per
  service run), `POST /api/drives/scan` reports `result: "setup_incomplete"` per drive (and at top
  level, with a message the Dashboard shows) and `POST /api/drives/rescan` returns 409. After the wizard,
  use **Check for New Media** or reinsert the disc (the wizard's last step says so).
- **Auto-rip toggle (`behavior.auto_rip_on_insert`, WP5).** When `true` (default) a disc insertion
  auto-starts a job as before. When `false`, an **automatic** insertion (the drive monitor event, the
  startup scan) does **not** start a job -- the drive is simply noted and the user starts the rip from
  the dashboard. The distinction is `JobEngine::onDiscInserted(drivePath, userInitiated)`: the drive
  monitor and startup scan pass `userInitiated=false` (gated by the toggle); **Check for New Media**
  (`scanForNewMedia`) and the manual `POST /api/drives/rescan` pass `userInitiated=true`, so a deliberate
  user action always starts the job even with auto-rip off. The skip is logged once per service run.
  Cancel/lock semantics are unchanged (nothing is reserved on the skipped path). Still gated by
  `setup.completed` first. Added on startup by `Db` for existing DBs; chosen in the setup wizard.
- **Restart recovery.** On startup every job in a status whose pipeline thread cannot survive a
  restart (pending, assigned, identifying, checking_nas, ripping, transcoding, verifying, and
  transferring *without* a `pending_transfers` row) is marked **failed** with "Interrupted (the
  service restarted) - reinsert the disc or press Retry". Untouched: needs_review and
  duplicate_review (their drive lock is restored so the startup scan does not open a duplicate job),
  and transferring jobs with a `pending_transfers` row (the tray copies those).
- **Retry** (`POST /api/jobs/{id}/retry`, failed or cancelled jobs; Jobs page and Job detail). Re-runs the
  **same job row** (`retry_count` + 1) rather than creating a new one as the server edition does:
  the working folder `rip_dir\job_<id>` belongs to the job id, the user stays on the job they
  clicked, and `retry_count / max_retries` on Job detail becomes meaningful (not enforced). Three modes:
  *copy* -- the finished files are still in the working folder (the NAS copy failed, was cancelled,
  or was interrupted): taken from the job's `pending_transfers` row (now kept when a copy fails),
  else rebuilt from the working folder. Only output that passed verification **in the current
  attempt** counts (an attempt starts at the job's latest `assigned` log entry -- a new insertion or a
  "read the disc again" retry), and the folder is checked again (format, no empty files, not fewer
  tracks than the cue lists). Copied directly if the service can reach the NAS -- the row is first
  taken off the tray's queue so a running tray cannot copy, complete or fail it concurrently -- else
  re-queued for the tray. No disc needed.
  *convert* -- an MP3 job whose FLAC rip finished in this attempt but whose conversion failed or was
  cancelled, with the FLACs still intact: only the conversion (then verify/copy) runs again. No disc.
  *pipeline* -- otherwise, if the job's drive is idle and holds a disc, the disc is read again
  (fresh placeholder disc row, merged by fingerprint as for a new insert; a previous attempt's
  unverified output is deleted, verified output is moved to `job_<id>_before_retry_<n>`).
  409 with a plain message otherwise: "Insert the disc and press Retry", drive busy (names the job),
  setup not complete, or not failed/cancelled. Every non-terminal job shows **Cancel** on Job detail
  (pending/identifying/checking_nas had none). Retry/Cancel buttons are disabled while their request
  is in flight. Mark-Done stays hidden in standalone.
- **Cancel stops the pipeline.** A job's thread owns a cancel flag from the moment it starts
  (insertion, approve, resolve-duplicate, retry). Cancel raises it; the thread checks it after
  identification, after the metadata lookup, before checking the NAS, before/after rip, conversion,
  transcode and copy (the tools themselves stop on it too) and ends the job **cancelled** -- a user
  cancel is never reported as *failed*. The drive stays locked until the thread has really ended, so
  Check for New Media cannot start a second job on a drive that is still being read. A job parked in
  needs_review/duplicate_review/waiting-for-tray has no thread and is cancelled directly. A late
  result (copy finishing, tray report) never changes a cancelled or done job. ~JobEngine waits up to
  5 s for running pipeline threads.
- **Service stop / crash never orphans a tool.** Every external program (cyanrip, ffmpeg,
  makemkvcon, HandBrakeCLI, the ffmpeg `-encoders` probe) is started suspended and put in one
  process-wide job object with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` (`client/src/ripper/ChildProcessJob.h`),
  so it dies with the service even if the service crashes or is killed. On a normal stop
  `JobEngine::shutdown()` raises every cancel flag, kills the tools at once
  (`TerminateJobObject` -- a read loop only sees its cancel flag when the tool prints something) and
  waits up to 20 s for the pipeline threads; jobs stopped this way keep their status and are
  recovered as "Interrupted" on the next start. If a thread is still inside an uninterruptible step
  after 20 s, `main.cpp` leaves the engine/Db allocated for process exit instead of freeing them under
  it. Before, cyanrip kept reading the drive after the service had stopped (the next start found the
  drive busy, Retry could not clear the locked working folder, and an upgrade would hit a locked exe).
- **A pipeline thread can never leave a job stuck.** If an exception escapes a thread (even
  `finishJob` throwing), `JobEngine::launch()` marks the job failed and always releases its drive and
  cancel flag.- **MP3 needs FFmpeg up front.** An MP3 job fails *before* ripping ("... FFmpeg ... was not found or
  cannot write MP3 ... The disc was not ripped.") instead of after an 8-minute rip.
- **Text and paths are UTF-8.** Every path string in JobEngine, CyanRipRipper, FlacToMp3Converter,
  NasCopy and (with `Input::utf8Paths`) MediaOrganizer is UTF-8 (`pathFromUtf8`/`pathToUtf8`);
  `path::string()` (ANSI) is only used at the boundary to the shared MakeMKV/HandBrake wrappers, which
  take native paths. Accented artist folders used to put invalid UTF-8 into job logs and make every
  `/api/jobs` call a 500; CJK names threw "No mapping for the Unicode character". Backstops: job log
  text is sanitised to valid UTF-8, and the REST API serialises with `error_handler_t::replace`.
- **Long paths.** Directory walks use `increment(ec)` (an over-long path is an error, not an
  exception/HTTP 500) and the service, tray and test exes embed a `longPathAware` manifest
  (`windows-standalone/longpath.manifest`; effective when Windows' LongPathsEnabled policy is on).
- **MakeMKV key problems.** `MakeMkvRipper` watches makemkvcon's robot-mode `MSG:` lines for its
  licensing messages (code 5021 "This application version is too old ... enter a registration key",
  "Evaluation period has expired ... activation key", "registration key"/"beta key"/"key ... invalid/
  expired" text). If the rip then fails, the job fails with: "MakeMKV's beta key has expired or is
  missing. Get the current key at https://forum.makemkv.com/forum/viewtopic.php?t=1053 and enter it
  in MakeMKV (Help > Register)." (Otherwise a failed rip now includes MakeMKV's last message.) A
  licensing reminder on a *successful* rip (DVDs stay free) never fails the job.

## Configuration backup, About / Licensing, diagnostics (WP3, standalone-only)
These are gated on `useAuth().standalone`; the server edition UI is unchanged.

- **Export / Import configuration** (two buttons at the top of Settings).
  `GET /api/config/export` returns the user's *configuration* as JSON so it
  survives a DB rebuild: every non-hidden `settings` row (API keys, `tools.*`
  paths, `audio.*`, `naming.media_server`, etc.), NAS paths and quality
  profiles. It is NOT job/library/disc/metadata-cache history (a larger "data"
  export is a future item). The file includes the **real secret API keys** on
  purpose (the point of a restore) and carries `_warning` + `schema`/`version`/
  `app_version`; the UI warns the user the file holds their keys.
  `POST /api/config/import` takes `{ mode: "merge"|"replace", config }` (mode
  also a `?mode=` param, or the bundle posted directly). Validates defensively
  (unknown setting keys ignored, non-string values -> 400, bad body -> 400,
  never crashes). Settings are UPDATE-only on existing non-hidden/non-readonly
  rows; **`setup.completed` is never clobbered**. NAS paths match by **label**
  and profiles by **name** (re-import updates, never duplicates). `replace`
  additionally drops NAS paths/profiles not in the bundle that no job
  references. Returns per-category counts.
- **`GET /api/system`** (standalone-only): OS name/version/build (from the
  registry -- `GetVersionEx` lies without a manifest), arch, logical CPU count,
  total RAM, and the app's own data/install/log locations. No secrets, no user
  names.
- **About page** (`/about`, standalone nav): app name + version (`/health`),
  dashboard URL, data locations + tool status (`/api/system`, `/api/tools`),
  basic system info, the disclaimer, a Licensing link and a diagnostics button.
- **Acknowledgements & Licensing page** (`/licensing`, standalone nav): renders
  the EULA + disclaimer (from `web/src/legal.ts`, mirroring `legal/LICENSE.txt`)
  and a notice for every third-party component with its licence, linking the
  full licence texts shipped under `<install>\web\legal\third-party\`.
- **First-run disclaimer** (dashboard, standalone): a modal shown until the user
  clicks "I understand"; the acknowledgement persists in the `ui.disclaimer_ack`
  settings row (added on startup by `Db`), so it carries across browsers/machines
  for the install. It only mounts on the Dashboard, which renders after the
  setup wizard, so it never fights the wizard.
- **Diagnostics** (`GET /api/diagnostics`): a `.zip` for bug reports --
  `README.txt`, `system-info.json`, `versions.json`, `config-redacted.json`
  (the config export with **all** API keys/tokens **and** contact-identifying
  `api.*` values -- notably `api.musicbrainz.useragent`, which embeds a contact
  email -- shown as `***set***`/`(not set)`; the full config export keeps them
  verbatim so a restore works),
  `jobs-recent.json` (id/status/timestamps + latest log line, no secrets) and
  the service `logs/`. Built with the dependency-free `service/src/MiniZip.h`
  (stored entries, CRC-32 -- the service links no zip library).

### Server-only pages excluded from the standalone bundle (WP3)
The standalone web build is produced with `npm run build:standalone`
(`vite build --mode standalone`, which loads `web/.env.standalone` ->
`VITE_EDITION=standalone`). In `web/src/App.tsx` the server-edition-only pages
(Clients, Users, Roles, Notifications, Workflows, Maker Portal, Data) are
lazy-loaded behind a `IS_STANDALONE ? null : lazy(() => import(...))` ternary.
Vite inlines `VITE_EDITION` as a literal, so in the standalone build each
`import()` is in a provably-dead branch that Rollup drops -- the page source
never enters the emitted JS (verified by grepping the built bundle). The server
build leaves `VITE_EDITION` undefined, so every page is present (as its own lazy
chunk). The runtime `standaloneHidden` nav gating in `Layout.tsx` is unchanged;
`App.tsx`'s exclusion list must be kept in sync with it. About/Licensing routes
exist in both builds but are reachable only via the standalone nav.

## First-run setup wizard (WP5, standalone)
A plain-language, eight-step wizard anyone can follow (the React component gates on
`useAuth().standalone`; the **server edition wizard is unchanged**). Each step has a short, collapsible
**"Why do I need this?"** help line, a progress indicator and Back/Next. Steps:
1. **Welcome** -- what the app does in one paragraph; notes the WP3 first-run disclaimer was accepted at
   install (it is *not* duplicated here -- that modal only mounts on the Dashboard).
2. **Disc drive** -- confirms the optical drive(s) from `GET /api/drives` (friendly "Disc drive (F:)"),
   with a Re-check button and a friendly message when none is found.
3. **Ripping programs** -- MakeMKV / HandBrake / cyanrip / FFmpeg status from `GET /api/tools` (reuses
   `ToolsPanel`): MakeMKV needed only for DVD/Blu-ray, cyanrip + FFmpeg shown as **Included**, audio CDs
   work without MakeMKV. Download + Re-check.
4. **Where your media goes** -- the key fix (defect #2). Auto-detects candidate destinations and, for each
   one kept, the user picks a **type** (Movies/TV/Music -> `nas_paths.content_type` movie/tv/audio) and a
   label, with a **Test access** button. Handles multiple destinations, paths with spaces, UNC paths and
   the bare-drive-letter case (normalised to `X:\` by `normalizeNasPath`). At least one is recommended;
   skipping is allowed with a warning. **Mapped-drive detection approach:** see the storage relay below.
5. **How to rip** -- video quality (a dropdown over the video quality profiles, `set-default`), music
   format FLAC vs MP3 (`set-default` on `Audio-FLAC`/`Audio-MP3`), media-server naming Plex /
   Jellyfin-Emby / Generic (wires `naming.media_server`, previously API-only), and the
   **auto-rip-on-insert** toggle (`behavior.auto_rip_on_insert`).
6. **Disc information** -- optional API keys: TMDB, **TheTVDB**, OMDb, MusicBrainz user-agent, **Discogs**
   (TheTVDB and Discogs were missing before -- June findings #6), each with a "how to get this key" link;
   music works with no key. Saved via the settings API.
7. **Review** -- a plain summary of every choice with per-row **Edit** links.
8. **All set** -- "You're all set", insert-a-disc guidance, `POST /api/setup/complete`.

**Re-runnable from Settings:** a standalone-only **Re-run setup** button routes to `/setup?rerun=1`. In
re-run mode the wizard does not redirect away even though `setup.completed` is true, shows a "Re-running
setup" badge, and **pre-fills from the current settings / nas_paths / profiles** -- existing data is kept,
never wiped, and `setup.completed` is never reset.

### Storage detection relay (how mapped drives are found)
The service is **LocalSystem / Session 0** and cannot see the user's mapped drive letters, their NAS logon
or their "Network Shortcuts" (`shell:NetHood`) -- the same split as "Copying to the NAS" above. So both the
destination list and the **Test access** check run in the **tray** (user session) and are relayed through
the service's in-memory endpoints:
- `GET /api/storage/suggestions` -> `{ source, tray_seen, updated_at, candidates:[{path,label,kind,accessible,writable}] }`.
  Returns the tray's last-pushed list; **falls back** to the drives the service itself can see (so the
  wizard always shows something) with `accessible/writable` null.
- `POST /api/storage/suggestions` (tray) -- the candidates the tray enumerated in the user session
  (`tray/src/StorageProbe.*`: local fixed + mapped drives via `GetLogicalDrives`/`GetDriveType`, plus
  resolved NetHood UNC shortcuts via COM `IShellLink`). The tray's `TransferWorker` pushes roughly every
  30 s.
- `POST /api/storage/test {path}` -> `{id}`; `GET /api/storage/test/{id}` -> `{status,reachable,writable,message,checked_by}`.
  The tray answers via `GET /api/storage/test-requests` + `POST /api/storage/test-requests/{id}`
  (`checked_by:"tray"`). If the tray does not answer within ~2.5 s the **service** answers itself
  (`checked_by:"service"`, noting mapped drives are only visible to the tray) so the wizard never hangs.
  Manual entry is always available regardless.

Why this approach (option (a) from the brief): the tray already runs in the user session and already polls
the service (`TransferWorker`), so pushing suggestions / answering test requests on that existing loop fits
the established tray->service split with no new process or protocol; the browser cannot reach the tray
directly (the tray is an HTTP *client*, not a server).

New settings added on startup by `Db` for existing DBs: `behavior.auto_rip_on_insert` (default `true`).

## Version
`windows-standalone/VERSION` is the only place the version is written. CMake reads it into the
`RIPNETMEDIA_VERSION` compile definition (service + tray; `GET /health` returns `"version"`, the service
log's startup line includes it) and `build-standalone-installer.ps1` passes it to makensis as
`/DVERSION=...` (branding text, registry `Version` and Add/Remove Programs `DisplayVersion`).

## Build and package

Windows only. Toolchain used here: VS 2019 Build Tools + vcpkg (`x64-windows-static`)
+ Ninja; existing build dir `build-task9-standalone` (`-DBUILD_STANDALONE=ON`).

```powershell
# 1. From an MSVC developer environment (vcvars64.bat):
cmake --build build-task9-standalone
# 2. Web dashboard (outputs to server/web):
cd web; npm ci; npm run build
# 3. Installer (needs NSIS 3.x and dist\tools\{cyanrip-win64.exe, ffmpeg.exe, ffmpeg-LICENSE.txt}):
packaging\windows\build-standalone-installer.ps1      # -> dist\RipNetMedia-Standalone-Setup.exe
```

The script stages the exes, `db\schema.sqlite.sql`, `web\`, `tools\cyanrip-win64.exe`,
`tools\ffmpeg.exe` + `tools\ffmpeg-LICENSE.txt`, the notices and `LICENSE.txt` (staged next to
the `.nsi` so the licence page can embed it), then runs `makensis /DVERSION=<VERSION>`.
(FFmpeg makes the installer ~65 MB instead of ~9 MB.) The installer registers/starts the service
via the service exe's own `--install/--start` verbs, and its finish page tells the
user to right-click the tray icon → **Open Dashboard**.

Installer wizard pages: **Welcome** (with a brief, non-alarming SmartScreen note -- the app is
not yet code-signed, so Windows may show "Windows protected your PC -- More info -> Run anyway" on
first launch; actual signing is deferred) -> **Licence** (`MUI_PAGE_LICENSE` showing
`legal/LICENSE.txt`, the EULA + DISCLAIMER; must be accepted to continue) -> **Directory** ->
**Install** -> **Finish** (tray instructions + a pointer to the dashboard's About / Licensing pages).

First-upgrade tool-lock fix (D2): on install/upgrade, after `--stop`/`--uninstall` and before the
`tools\*` `File` lines, the installer kills only `cyanrip-win64.exe`/`ffmpeg.exe` processes whose
on-disk image lives **under `$INSTDIR\tools`** (a `Get-CimInstance` query filtered on
`ExecutablePath`, case-insensitive prefix). This clears an orphan left by an OLD service build
(from before the kill-on-job-close job object) that could otherwise hold a locked exe and fail the
overwrite; a user's own ffmpeg elsewhere is never touched. `SetOverwrite try` around the tools
`File` lines is a backstop (restored to `on` afterward so the main exes still fail loudly).

## Tests (headless, no hardware needed)
Run from `build-task9-standalone\bin`; each takes a DB path (and a port). The build copies
`dist\tools\{cyanrip-win64.exe, ffmpeg.exe, ffmpeg-LICENSE.txt}` into `bin\tools\` (target
`ripnetmedia-copy-tools`), so tests and an isolated service run from `bin` use the shipped tools.

| Executable | Covers |
|---|---|
| `ripnetmedia-service-jobenginetest.exe <db>` | Pipeline, drive lock, non-optical guard, audio-CD path, MusicBrainz disc-ID known answer, tool-path cleanup, NAS drive-letter normalisation, MakeMKV key detection, FLAC->MP3 (helpers, a real ffmpeg conversion if `ffmpeg.exe` is in `bin\tools\` or on PATH -- otherwise skipped with a NOTE, and the MP3 pipeline path), setup gate, restart recovery, Retry (pipeline / copy via tray / direct copy / convert / refusals), accented + CJK artist names end to end, cancel while identifying / ripping / converting, attempt-scoped copy retry, >260-char paths, direct copy taken off the tray queue, FFmpeg fail-fast + libmp3lame check, shutdown() kills a silent tool and leaves the job for recovery, tools die with a crashed process (kill-on-close job), an escaping thread exception fails the job and frees the drive, NAS duplicate check via the tray and the service (review, edition, edition-also-taken, Replace, Cancel + Retry, Proceed, restart, multi-disc), the check never widens above the album/season (D1) and Replace deletes only the reviewed files with every refusal path (D2/D3), findExistingMedia scopes + Extras-only (D4) + isStrictlyInside, real-FFmpeg AlbumRetagger (D5: FLAC+MP3, IDs gone, other tags kept, quoting), edition naming for generic/plex/jellyfin + TV + album, edition sanitising/60-char limit and 409, library filename of an edition, reinsert leaves no clutter job, a done job always has its library row, **WP5** auto-rip toggle (off: automatic startup scan / drive-monitor insert start nothing and log once, Check for New Media still starts a job; on: an automatic insert starts a job again) |
| `ripnetmedia-service-restapitest.exe <db> <port>` | REST lifecycle actions, settings, loopback-only binding, /health version, retry route, setup-incomplete scan/rescan, NAS path normalisation on save, ffmpeg in /api/tools (and required with an MP3 default), accented + CJK names through /api/jobs, /logs, retry and /api/transfers/pending, invalid-UTF-8 backstop, naming.media_server validation, transfer conflict route (incl. files[]), edition resolve/preview, edition in /api/jobs and /api/library, duplicate_nas_files + ripped_files_waiting, a Replace pending transfer's reviewed_path/replace_files/check_scope, duplicate_nas_* cleared when done, cleanup folders_removed, **WP3**: config export shape + verbatim secret, config import (merge update, unknown key ignored, non-string rejected, setup.completed protected, NAS upsert-by-label no-duplicate, replace removes an unreferenced orphan, bad body/mode -> 400), /api/system fields + no username, /api/diagnostics is a valid zip with secrets redacted (incl. the musicbrainz user-agent contact email, while the full export keeps it verbatim), empty config/import body -> 400, **WP5** storage relay (suggestions fallback vs tray push + drive-letter normalisation, test request -> tray lists -> tray answers -> wizard reads done result, unknown test id 404) |
| `ripnetmedia-tray-logictest.exe <db> <port>` | TrayApi against a real in-process service; the real TransferWorker's NAS check (conflict, edition, Replace) and the nas_exists notification. `--transfer-once <port>` runs the real TransferWorker once against a running (isolated) service for manual QA |
| `ripnetmedia-service-dbtest.exe <db>` | Db layer |

Fixtures use `FakeDriveMonitor` (fake drive letters would otherwise be rejected by
the real `GetDriveTypeA` guard). Real-hardware checks (audio CD end to end) were done
manually against an isolated service instance on another port with a temp DB. The NAS duplicate check
(2026-10-17, Iggy Pop *Lust for Life*): NAS = a temp folder renamed away while the service decides and
back for `ripnetmedia-tray-logictest.exe --transfer-once <port>` (the real TransferWorker); rip 1 copied,
rip 2 -> conflict (9 files) -> "Live" from the dashboard -> `Iggy Pop\Lust for Life (Live)`; rip 3 -> conflict
-> Cancel (files kept, NAS untouched) -> Retry -> conflict again -> Replace (stale track deleted, artwork kept).
QA re-check 2026-10-17 (D1-D5, after the Replace-scope fix): planted `Iggy Pop\Other Album\1 - precious.flac`
and, after the review, added `99 - Added After Review.flac` to the album, then chose Replace -> exactly the 9
reviewed FLACs were deleted and re-copied; the precious file, the added-later file, the `.cue` and `.log` all
survived (job log "deleted 9 reviewed media file(s)"). "Live" edition on a real rip -> the copied FLAC's
`album` tag read back as `Lust for Life (Live)` with the MusicBrainz album/disc IDs removed (FFmpeg
`-f ffmetadata`), and the original `Lust for Life` album was left untouched.

## Maintenance
Drive detection: `WindowsDriveMonitor::hasDiscPresent` (shared `client/src/drive`) opens the drive with
`GENERIC_READ`; `IOCTL_STORAGE_CHECK_VERIFY` needs read access, and opening with 0 access always failed
(so "Check for New Media" and the startup scan reported "no disc"). The duplicate-review buttons send the
web dashboard's `{action, mode, library_id, edition}` shape; `POST /api/jobs/{id}/resolve-duplicate` accepts that
and the tray/test `{proceed, replace_library_id, edition}` shape.

The Dashboard has a standalone-only **Check for New Media** button (`POST /api/drives/scan`): for each optical drive it starts a job if a disc is present and the drive is idle, and reports per drive (job started / busy / no disc). Empty drives never create a job (unlike the older `/api/drives/rescan`, which trusts the caller). A disc that was already ripped will start a job that lands in duplicate review.

The Jobs page has a standalone-only **Cleanup Failed Jobs** button (`DELETE /api/jobs/failed`): it removes
failed and cancelled job records and their logs (never jobs that have a library entry). It does **not** delete
files in the working folder - after a failed NAS copy those may be the only copy. With
`?remove_duplicate_files=1` (a default-off checkbox in the dialog) it also deletes the `rip_dir\job_<id>`
working folder of jobs that were **cancelled in a NAS Duplicate Review** -- those files are already on the NAS.
The response is `{deleted, folders_removed}`.

`dist\tools\ripnetmedia-cleandb.exe <db>` (source: `tools/cleandb.cpp`) clears job
history, discs, library and metadata cache while keeping settings/API keys, NAS
paths and quality profiles, and resets `setup.completed` so the wizard reruns.
`%ProgramData%\RipNetMedia` is writable only by the service/admin, so stop the
service and run it from an elevated prompt:
`Stop-Service RipNetMediaService; ripnetmedia-cleandb.exe "$env:ProgramData\RipNetMedia\ripnetmedia.sqlite"; Start-Service RipNetMediaService`.

## Decisions and known gaps
- **Release prep (WP4):**
  - **Autostart is tray-self-managed (#10):** the installer no longer writes the HKCU Run key;
    the tray sets it up for the real user on first launch and the menu toggle is authoritative
    (see "Tray autostart" above). Fixes autostart landing in an admin's profile.
  - **Installer licence page:** `MUI_PAGE_LICENSE` (EULA + disclaimer) must be accepted before install.
  - **First-upgrade tool-lock (D2):** orphan `cyanrip`/`ffmpeg` under `$INSTDIR\tools` are killed
    before the tools are overwritten (`SetOverwrite try` backstop). A user's own ffmpeg is untouched.
  - **SmartScreen:** welcome/finish text and this wiki note that the app is not yet code-signed, so
    "Windows protected your PC -> More info -> Run anyway" may appear on first launch. **Code signing
    is deferred** (user's call) -- still TODO before a wide public release.
  - **Diagnostics redaction:** `config-redacted.json` now also redacts `api.musicbrainz.useragent`
    (holds a contact email) and any contact/email-ish `api.*` value. Empty `POST /api/config/import`
    body is now a 400, not a silent 200 no-op.
- **Server-only by decision:** Maker Portal / Workflows (the standalone engine is a fixed
  pipeline and never reads the workflow tables), Notifications, Clients, Users/Roles, Data
  export/import. Hidden in standalone.
- **Hidden until built:** Mark-Done on jobs (no manual-complete in `JobEngine`). Retry is built (see above).
- **Retry reuses the job row** (server edition creates a new job) -- see "Job lifecycle rules".
- **MakeMKV message codes:** only 5021 is matched by code (from MakeMKV's documented robot output);
  the other licensing messages are matched by text, because their codes vary by version. Not
  verified against a real expired key on this machine.
- **Audio track names** come from cyanrip's `{track}`, which is not zero-padded (`1 - Lust for Life`),
  so file managers sort track 10 before track 2 on 10+ track discs. Unchanged by the MP3 work.
- **Cover art** is not embedded (cyanrip's static build can't follow the https redirect). Non-fatal.
- **Licensing (WP3):** `THIRD-PARTY-NOTICES.txt` is now a full long-form notice (cyanrip LGPL-2.1,
  FFmpeg LGPL-3.0, MakeMKV proprietary/not bundled, HandBrake GPL-2.0, and the metadata providers
  with attribution), and the in-app `/licensing` page renders the EULA + per-component notices. Full
  licence-text bodies live under `legal/third-party/` and ship to `<install>\web\legal\`. **TODO for
  the user before public release:** `legal/third-party/LGPL-2.1.txt` and `GPL-2.0.txt` are
  placeholders pointing at the canonical gnu.org URLs -- paste the verbatim texts in (not fabricated
  here to avoid an inexact copy). `LGPL-3.0.txt` is the real text (copied from FFmpeg's). Still
  review cyanrip's exact terms (it statically includes FFmpeg) and keep the LGPL corresponding
  source for the written offer; have the EULA reviewed by a lawyer.
- **Startup race (fixed):** a disc in the drive is no longer ripped before setup is complete.
- Tool status and profile defaults are read live; the drive monitor and API port are not.
- **Editions (WP2):** audio editions re-tag the ALBUM tag and drop the release MusicBrainz IDs; video
  editions change folder/file names only (Plex/Jellyfin match movies by those names, not tags). TV and music
  edition conventions have no official Plex/Jellyfin equivalent (see "Editions and media-server naming");
  none of the three conventions was checked against a running media server here. The setup wizard's
  **How to rip** step now lets the user choose `naming.media_server` (WP5); Settings also has the dropdown.
- **NAS check limits:** the pre-rip check (service-visible NAS) only predicts the default folder template
  `{title} ({year})`; other templates are only checked after the rip. A full-disc rip is pre-checked against
  `Title (Year)`, not `Title (Year) [DISC]`. **Replace** deletes only the media files the user was shown in
  the review (never folders/artwork/cue/log, never a parent scope, never a file added after the review); there
  is no recycle bin on a network share.
- **Movie extras discs** are now copied to `NAS\Title (Year)\Extras` (they used to land in `NAS\Extras`).
- **Library row before done (D1):** `finishJob` writes the output path and library row, then marks the job
  done, so a poll never sees a done job without its Library entry.
