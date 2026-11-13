# spotibackup web UI

The controller serves this UI on port 8080. It's how you set up, choose, schedule and watch backups. Architecture is in [design.md](design.md).

The screens are drawn in Pencil, in `docs/ui-design/spotibackup.pen`. This document is the source those screens are drawn from. Where they differ, this document wins until it's updated.

**Status:** Phase 1 draft, awaiting review.
- **Shaped with the user:** every page: Setup wizard, Library, Albums, Dashboard, Schedule, History, Settings and Player.
- **Not drawn yet:** the Pencil screens.
- **Not reviewed yet:** the Dashboard's Full backup card, and the backup-length line in wizard step 8.

## Design language

- **Model:** the current Spotify desktop client's design language: dark surfaces, one green accent, dense rows, rounded cards and a left sidebar. It's a reference, not a copy.
- **Our own work:** no Spotify font, icon, logo or image is copied. Components are rebuilt in Pencil from the client's styling: colour, spacing, radius, type scale and component anatomy.
- **Icons:** [Lucide](https://lucide.dev).
- **Font:** Figtree for now (free, close to the client's geometric sans). It can change once it's been seen in the screens.
- **Component set** in the `.pen` file:
  - tokens: colour, type, spacing, radius;
  - controls: primary, secondary, icon and split buttons, text input, search input, toggle, checkbox row, filter chip, week-grid cell (active, inactive, hover);
  - navigation: sidebar nav, stepper;
  - lists and cards: list row (playlist, album or track), card with corner badge, modal;
  - feedback: progress bar, level meter, status pill, toast, inline alert.

## Navigation

- **Before setup:** until setup is complete, every route redirects to the Setup wizard.
- **After setup:** a left sidebar (Spotify's Home / Library pattern) with:
  - Dashboard
  - Library (playlists and Liked Songs)
  - Albums
  - History (with Runs, Tracks and Logs tabs)
  - Schedule
  - Settings
  - Player
- **Bottom bar:** while a run is going, a persistent bar like the client's now-playing bar shows the current track, item progress and a Stop button. It links to the Dashboard. On the Dashboard itself, the same strip sits at the top instead.

- **Expiry banner:** 4 weeks before the Web API connection expires, a yellow banner appears at the top: "Spotify connection expires on <date>", with a Reconnect button. Once it has expired, the banner turns red and scheduled runs are held.

Schedule is its own page. Logs is a tab inside History.

---

## Setup wizard

A linear stepper, styled like the client's onboarding: dark background, a centred card and a green primary button.

- **Progress:** each step's state is stored in `/config`. Reloading the page, or restarting the container, resumes at the first unfinished step.
- **Navigation:** completed steps can be reopened from the stepper.
- **Re-running:** Settings → About → "Run setup again" reopens the wizard. The Terms notice does not show again.

### 1. Welcome and Terms notice
- **Content:**
  - What spotibackup does, in two lines.
  - The notice: recording breaks Spotify's terms of use, and accounts using recording software have been suspended before. It suggests using a separate Premium account. Whether private copies are legal depends on the country.
- **Action:** a checkbox, "I understand", then Continue.
- **After acknowledging:** the acknowledgement is stored with a timestamp, and this screen never shows again. A later "Run setup again" starts at step 2.

### 2. Spotify client
On first start, the container installs the Spotify Linux client from Spotify's apt repository into `/config`. This step shows that install.
- **States:**
  - **Installing:** a progress bar with log lines.
  - **Installed:** shows the version, with a Continue button.
  - **Failed:** shows the error and the last log lines, with a Retry button.
- If the client is already installed, the step is ticked off on its own.

### 3. Log in to Spotify
The prototype found that the Linux client (1.3.4) logs in by QR code. Its login screen shows "Scan code to log in", which you scan with the Spotify phone app. The client's other button, "Log in", opens an external browser, and the image has none, so the wizard never uses it.

- **Layout:**
  - **Left:** an embedded noVNC view of the client's login screen, so the QR code is visible.
  - **Right:** the steps:
    1. Open Spotify on your phone.
    2. Scan the code.
    3. Approve the login. Use the account whose playlists you want to back up.
- **Completion:**
  - The controller polls for login, and the step turns green by itself. There is no "I'm logged in" button.
  - Detection works through MPRIS, confirmed working in the prototype.
- **Code expired, or the client showing something else:** a "Reset login screen" button restarts the client.
- **Larger view:** the same view is available full size on the Player page.

### 4. Spotify client settings
Spytify records whatever the player outputs, so anything Spotify does to the sound ends up in the files. You change these once, in the client's Settings, using the noVNC view embedded on the left of this step. The checklist sits on the right.

Rows are grouped as they are in the client's Settings. Each row shows the required value, a one-line reason, and a pill for how it is checked:
- **Auto:** the controller reads it.
- **Confirm:** you tick it.
- **Set for you:** the controller sets it.

| Section | Setting | Required | Why | Check |
|---|---|---|---|---|
| Playback | Crossfade | Off | Overlapping tracks can't be split. | Auto (Spytify `core.info` warning) |
| Playback | Gapless playback | Off | A short gap gives clean cuts. Spytify trims the silence. | Auto (Spytify `core.info` warning) |
| Playback | Automix | Off | Blends and alters transitions. | Auto (prefs `audio.automix`) |
| Playback | Autoplay | Off | Stops similar songs from playing after the playlist. It may be hidden for some accounts. **Not a blocker:** the controller also stops when the playing context changes. | Confirm |
| Playback | Volume normalization | Off | Alters levels, and "Loud" compresses. Use ReplayGain at playback instead. | Auto (prefs `audio.normalize_v2`) |
| Playback | Equalizer | Off | Colours the recording. | Confirm |
| Playback | Mono audio | Off | Would record mono. | Confirm |
| Audio quality | Streaming quality | Very high (the highest offered) | It is the quality that gets recorded. | Confirm |
| Audio quality | Auto adjust quality | Off | Spotify buffers instead of silently lowering quality. | Confirm |
| Player | Volume | 100% | Spytify marks takes below full volume as "(low volume)" and doesn't count them. | Set for you: `player.control full_volume` before every run |

- **Notes:**
  - "Auto" rows read Spotify's local prefs file (`/config/home/.config/spotify/Users/*-user/prefs`) or Spytify's `core.info`. Crossfade and gapless aren't in the prefs file because they're stored on Spotify's servers. Spytify's warning "turn crossfade and gapless playback off in the player" covers them.
  - Pill states: OK, Needs change, or Unknown (it can't be read).
  - **Re-check** reads every Auto row again.
- **Continue:** allowed when every Auto row is OK and every Confirm row is ticked. Autoplay can be left unticked, with a note.
- **After setup:** the same checklist appears in Settings → Spotify client. Before each run, the run engine re-checks the Auto rows. A row that has drifted fails the run with a clear message (see the Failure flow).

### 5. Connect the Spotify Web API
spotibackup uses the Web API to list your playlists and albums and to start playback on the container's client. This needs a Spotify developer app that you own, which requires Premium.

1. **Instructions:** open the Spotify developer dashboard and create an app. It shows the redirect URI to paste into the app, `http://127.0.0.1:8888/callback`, with a copy button.
2. **Client ID:** paste it into a field.
3. **Authorize:**
   - Opens Spotify's consent page in a new tab.
   - After you approve, the browser goes to `http://127.0.0.1:8888/callback?...` and shows an error page. That's expected.
   - You copy that whole URL from the address bar and paste it into the field this step then shows. Then Connect.
4. **Checks:** both must pass for the step to turn green.
   - The token works. Shows "Connected as <display name>".
   - The container's client is listed as a playback device. If not: "Make sure the client in step 3 is open and logged in", with Retry.

Scopes requested:
- `playlist-read-private`, `playlist-read-collaborative`
- `user-library-read`
- `user-read-playback-state`, `user-modify-playback-state`

**Token storage and expiry:**
- The token is kept in `/config` and refreshed by the controller. The Client ID is permanent unless the app is deleted.
- **Six-month expiry:** since mid-2026, a Spotify refresh token expires 6 months after you authorize, and refreshing does not extend it. An expired token gets `400 invalid_grant`. So the controller stores the authorization date, and the UI shows when the connection expires.
- **Connection summary:** the step's success state shows "Connected as <name> · expires <date>".

**Reconnect:**
- This same step, opened on its own from Settings or from the expiry banner, is the Reconnect flow.
- It skips the Client ID if one is already stored, and goes straight to Authorize, paste, check.
- Reconnecting resets the 6-month clock.

### 6. Output
- **Music folder check:** confirms `/music` is a mounted volume and writable, and shows the free space.
  - If it isn't, a red alert explains how to map `/music` to a shared folder in the UGOS Docker app, with a Re-check button.
- **Format:** the prototype found that the Linux client (1.3.4) offers "Very high" (Ogg Vorbis 320 kbps) at most, with no Lossless. The source is always lossy, so:
  - **Offered:**
    - AAC 256 (default)
    - AAC 320
    - MP3 320
  - **FLAC and WAV:** under a collapsed "More formats", each marked "No benefit: the source is lossy. Bigger files, same quality."
- **Folder layout:** fixed, shown as information, not as a choice: each track is saved once under Artist / Album, and each playlist gets an `.m3u` file in `Music/Playlists/`. It shows a preview path, for example `Music/Radiohead/OK Computer/Radiohead - Airbag.m4a`. That is Spytify's default name, AAC with tags and a 640 px cover. Spytify's own duplicate index and working files live in `Music/.spytify/`.
- Everything here can be changed later in Settings.

### 7. What to back up
Two large choice cards:
- **"Back up all my playlists"** (default):
  - Every playlist you created is tracked, and so are the ones you create later.
  - It shows how many there are now ("You have 23 playlists").
  - A checkbox below it, "Include Liked Songs", is on by default.
- **"I'll choose in the Library":** nothing is tracked yet. You pick playlists afterwards.

The choice sets the **"Automatically track playlists I create"** setting. You can change it later in the Library's options menu or in Settings.

Followed and other people's playlists are never tracked automatically. They are added with the Library's **+** button.

### 8. Done
- **Content:** a summary of the account, format, folder and what is tracked ("23 playlists + Liked Songs").
- **Backup length:** how long the first full backup takes, since recording is real time: "About 298 h of music to record". While the track lists load, it shows "Measuring…". Followed playlists add "+ 2 followed playlists (unknown length)". The figure comes from the [backlog estimate](design.md#backlog-estimate).
- **Primary:** "Open Library".
- **Secondary:** "Back up now". It starts a first run of everything tracked and opens the Dashboard.

### Setup endpoints

| Method | Path | Used by | Notes |
|---|---|---|---|
| GET | `/api/setup` | all steps | Each step's state (`todo`, `active`, `done`, `failed`), plus the first unfinished step. |
| POST | `/api/setup/tos` | 1 | Stores the acknowledgement and its timestamp. |
| GET | `/api/client/status` | 2, 3 | Client installed, its version, install progress and whether it's logged in. |
| POST | `/api/client/install` | 2 | Retries the install. |
| GET | `/api/client/checklist` | 4 | Each setting: `ok`, `needs_change` or `unknown`. |
| POST | `/api/setup/checklist/confirm` | 4 | Manual "I've set these". |
| WS | `/vnc` | 3, 4, Player | noVNC websockify proxy. |
| POST | `/api/client/restart` | 3 | Resets the login screen when the QR code has expired. |
| POST | `/api/spotify/app` | 5 | Stores the Client ID. |
| POST | `/api/spotify/auth/start` | 5 | Returns the authorize URL. The PKCE verifier stays on the server. |
| POST | `/api/spotify/auth/finish` | 5 | Body: `{redirect_url}`. Exchanges the code and returns the user's display name. |
| GET | `/api/spotify/devices` | 5 | Whether the container's client is a device. |
| GET | `/api/spotify/connection` | 5, banner, Settings | Display name, `authorized_at`, `expires_at` and state (`ok`, `expiring`, `expired`). |
| GET | `/api/storage` | 6 | Whether `/music` is mounted and writable, plus free bytes. |
| GET/PUT | `/api/settings` | 6, 7, Settings | Format, bitrate, folder layout and `auto_track_owned`. |
| GET | `/api/library/summary` | 7, 8 | Count of owned playlists, and whether Liked Songs is tracked. |
| GET | `/api/backlog` | 8 | The backup length. Shared with the Dashboard. |
| WS | `/api/events` | all | Live push. During setup it carries client-install progress and setup state changes. |

---

## Library

The Library is where you choose what gets backed up. It follows the client's own Library: a header with filters and a view switch, then the items.

### What's in it
- **By default:**
  - **Playlists you created:** they appear on their own, including new ones as you make them.
  - **Liked Songs:** pinned first, with the client's gradient-heart tile drawn in our own style. It's treated as a playlist.
- **Added by you:** through the **+** button.
  - Playlists you follow.
  - Any playlist pasted as a link.
- **Not supported:** playlists made by Spotify (Discover Weekly, Daily Mixes). They aren't available through the API, so a pasted link to one is refused with that message.

### Header
- **Title:** "Library", with a one-line summary: "18 of 23 tracked · last backup 03:12 today".
- **Filter chips** (like the client's): All · Tracked · Not tracked · Created by you · Added.
- **Search:** a box that filters by name.
- **Sort:** Recently backed up · Name · Recently added · Track count.
- **View switch:** List / Grid, like the client's Library view options. The choice is remembered per browser.
- **+ button:** adds playlists (see "Add playlists" below).
- **Options menu (⋯):**
  - "Automatically track playlists I create", a toggle;
  - "Back up all tracked now".

### Item states
Every row and card shows one status pill:

| Pill | Meaning |
|---|---|
| Not tracked | The toggle is off. Its files, if any, are kept. |
| Never backed up | Tracked, but no run has reached it yet. |
| Up to date | Every known track is recorded. |
| n new | Tracks added since the last backup. The next run records them. |
| Partial | Some tracks failed or were skipped, other than as duplicates. |
| Recording / Queued | Part of the current run. It links to the Dashboard. |
| Unavailable | The playlist was deleted, made private or unfollowed. Its files are kept. |

### List view
A Spotify-style table, one row per item:

| Column | Content |
|---|---|
| Toggle | Track / don't track (switch). |
| Cover + name | Cover thumbnail, name, and the owner below it ("by you", "by Lorde", "added by link"). |
| Tracks | "84 / 120 recorded". For followed or linked playlists, whose totals the API won't give, it's "32 recorded" with an info tooltip. |
| Last backup | Relative time ("2 days ago"), or "Never". |
| Status | The pill. |
| ⋯ | Row menu: Back up now · Open in Spotify · Show files path · Remove from Library (added items only). |

Hovering a row shows a play-style round green button, "Back up now", in the position of the client's play button.

Albums are not in the Library. They have their own page (see [Albums](#albums)), because they work differently.

### Grid view
- **Card:** a square cover, a name and a status pill.
- **Corner badge:**
  - Tracked: a filled green check.
  - Not tracked: a hollow circle. Clicking the badge toggles tracking.
- **Progress ring:** while an item is recording, a thin ring around the badge shows its progress.
- **Hover:** shows the same round "Back up now" button.

### Detail view
Clicking an item opens it in place, like a playlist page in the client.
- **Header:** a large cover, the name, the owner, the track count and the total duration, then the tracking toggle and a large "Back up now" button.
- **Track table:**

| Column | Content |
|---|---|
| # | Position, in the Web API order. This is also the `.m3u` order. |
| Title / artist | |
| Album | |
| Duration | |
| State | Recorded (with file size, and the file path on hover) · Skipped (duplicate) · Missing · Failed (reason) · New. |

- **Playlists without item access** (followed or linked, unless you collaborate on them): the table is built from what has been recorded so far, with the note "Spotify doesn't share this playlist's track list with the app; tracks appear here as they're recorded."
- **Footer:** "Remove from Library" for added items; "Stop tracking" for your own. Neither one deletes files.

### Add playlists (the + modal)
A modal in the client's dialog style.
- **Top:** a search box, "Search followed playlists or paste a Spotify link". It accepts `https://open.spotify.com/playlist/…` and `spotify:playlist:…`.
- **Typing text** filters the suggestions below it.
- **Pasting a link:**
  - The link resolves into a preview row (cover, name, owner) with an **Add** button.
  - If it can't be used, an inline error explains why: not found, private, or made by Spotify and not supported.
- **Suggestions:** the playlists you follow that aren't in the Library yet. Each row shows a cover, the name, the owner and an **Add** button.
  - The list shows a "Nothing to suggest" empty state.
- **Added items:** they come in tracked. The modal stays open so you can add several, and Done closes it.

### Library endpoints

| Method | Path | Notes |
|---|---|---|
| GET | `/api/library` | Items with these fields:<br>• `id`, `kind` (`owned`, `liked`, `followed` or `linked`)<br>• `name`, `owner`, `cover`<br>• `tracked`, `total` (null when unknown), `recorded`, `new`<br>• `last_backup_at`, `status`<br><br>It filters, searches and sorts on the server. |
| PATCH | `/api/library/{id}` | `{tracked}` |
| POST | `/api/library` | `{url}` adds a followed or linked playlist. It returns the item, or an error code: `not_found`, `private` or `spotify_owned`. |
| DELETE | `/api/library/{id}` | Removes an added item. The files are kept. |
| GET | `/api/library/suggestions?q=` | Followed playlists not yet in the Library. |
| POST | `/api/library/resolve` | `{url}` returns a preview without adding the playlist. |
| GET | `/api/library/{id}/tracks` | The track list with per-track state. |
| POST | `/api/runs` | `{items: [id…]}` "Back up now". Queues the items in a forced run. Shared with the Dashboard. |
| PUT | `/api/settings` | `auto_track_owned` |
| WS | `/api/events` | `library.item` updates: status, counts and progress while recording. |

---

## Albums

A separate sidebar page, because albums almost never change once they're out.
- **Playlists:** tracked, and re-checked on every run for new tracks.
- **Albums:** **added once and recorded once**. After a complete recording, an album is done, and scheduled runs skip it.

### What's in it
- **The page:** the albums you have added for recording, in a grid or a list.
- **Where albums come from:** your **saved albums** (`GET /me/albums`).
  - Albums you downloaded in a Spotify app are also saved albums, so they're included.
  - The Web API doesn't report which albums are downloaded, so there's no separate "downloaded" marker.
- **Nothing is added automatically:** you choose albums with the **+** button.

### Header
- **Title:** "Albums", with a one-line summary: "42 albums · 39 recorded · 3 queued".
- **Filter chips:** All · Recorded · Queued · Partial.
- **Search:** a box that filters by album or artist.
- **Sort:** Recently added · Album · Artist · Year.
- **View switch:** List / Grid, remembered separately from the Library's.
- **+ button:** in the header, to the right of the view switch. It opens "Add albums".

### Album states

| Pill | Meaning |
|---|---|
| Queued | Added, not recorded yet. The next run (scheduled or forced) records it. |
| Recording | Part of the current run. It links to the Dashboard. |
| Recorded | Every track is recorded. Scheduled runs skip it from now on. |
| Partial | Some tracks failed. The next run retries only the missing tracks; the recorded ones are skipped as duplicates in under a second. |
| Unavailable | Spotify no longer offers the album. Its files are kept. |

### Grid view
- **Card:** a square cover, the album name, and the artist and year below it.
- **Status:** a small pill on the card. A Recorded album gets a green check badge instead.
- **Progress ring:** while the album is recording, a ring shows its progress.

### List view

| Column | Content |
|---|---|
| Cover + album | Cover thumbnail, album name, artist. |
| Year | |
| Tracks | "11 / 11". Album totals are always known. |
| Recorded | The date it was completed, or "—". |
| Status | The pill. |
| ⋯ | Row menu: Record again · Open in Spotify · Show files path · Remove from Albums. |

- **Record again:** re-queues the album. Tracks already recorded are skipped as duplicates, so only missing tracks are recorded.
- **Remove from Albums:** never deletes files.

### Detail view
The same layout as a playlist's detail view.
- **Header:** a large cover, the album, the artist, the year, the track count and the total duration, then a large "Record now" button.
- **Track table:** each track with its state: Recorded, Skipped (duplicate), Missing or Failed.

### Add albums (the + modal)
A modal in the same style as the Library's + modal.
- **Top:** a search box, "Filter by album name". It filters the list as you type.
- **List:** your saved albums that haven't been added yet. Each row shows a cover, the album name, the artist, the year, the track count and an **Add** button.
  - The list is ordered by most recently saved.
  - It's loaded fully when the modal opens, so filtering is instant.
- **Adding:** an added album leaves the list and joins the page as Queued. The modal stays open so you can add several, and Done closes it.
- **Empty state:** "All your saved albums are already added". It suggests saving albums in Spotify first.

### Album endpoints

| Method | Path | Notes |
|---|---|---|
| GET | `/api/albums` | The added albums, with these fields:<br>• `id`, `name`, `artist`, `year`, `cover`<br>• `total`, `recorded`, `status`, `completed_at`<br><br>It filters, searches and sorts on the server. |
| GET | `/api/albums/saved` | Saved albums not yet added, for the modal. The UI does the filtering. |
| POST | `/api/albums` | `{ids: [...]}` adds albums. They start Queued. |
| DELETE | `/api/albums/{id}` | Removes an album from the page. The files are kept. |
| GET | `/api/albums/{id}/tracks` | The track list with per-track state. |
| POST | `/api/runs` | `{items: [id…]}` "Record now" or "Record again". Shared with the Library and the Dashboard. |
| WS | `/api/events` | `album.item` updates: status and progress while recording. |

---

## Dashboard

The landing page after setup. It has two states: **idle** and **running**. A WebSocket switches it between them live.

### Idle
Three cards, stacked.

**1. Next backup**
- **Content:**
  - When the next run is: "Tonight 01:00–07:00 · in 9 h".
  - What it will cover: "18 playlists to check · 3 albums queued".
- **Buttons:** a large green **Run now** button. A quieter "Change schedule" link goes to Schedule.
- **Variants:**
  - **No schedule (manual only):** "No backup scheduled", with Run now.
  - **Held:** if the Web API connection has expired, or the client is logged out, the card turns red and says why. Its main button becomes Reconnect, or "Log in again" (wizard step 3), in place of Run now.
- **What Run now runs:** every tracked playlist, plus Queued and Partial albums.

**2. Full backup**
How far the backup of everything tracked has got, and how long the rest takes. The figures come from the [backlog estimate](design.md#backlog-estimate).
- **Progress bar:** recorded time out of total time, with "4,708 of 5,120 tracks · 87%" under it.
- **Time left:** "About 27 h of recording left".
- **Finish:** "Done around Sat 16 Nov at this schedule", from the schedule's active hours. With Manual only: "Run now to start; recording is real time".
- **Unknown items:** "+ 2 followed playlists (unknown length)", when there are any.
- **Freshness:** a quiet "Updated 01:00" line. The figures refresh when the page opens, at most once every 15 minutes, and after each item.
- **Variants:**
  - **Complete:** a green check and "Everything is backed up · 298 h of music".
  - **Measuring:** before the first track lists have loaded, a skeleton bar and "Measuring your library…".

**3. Last run**
- **When:** start time, duration and trigger (Scheduled or Run now).
- **Outcome pill:** Completed · Completed with issues · Window ended · Stopped · Failed.
- **Counters:** recorded · skipped (duplicate) · failed, and the size added ("+412 MB").
- **Issues:** up to three items that had problems, each with a one-line reason ("Chill Mix: 2 tracks failed: low volume").
- **Link:** "View in History".
- **Before the first run:** "No backups yet".

### Running: queue-first
The item queue is the main element. A compact now-playing strip sits above it.

**Run header**
- "Backing up · started 01:00 · 0:42 elapsed · Scheduled · stops by 07:00". The last part only appears for scheduled runs.
- Live counters: Recorded n · Skipped n · Failed n.
- Time left for the whole backlog, from the Full backup card: "about 26 h left".
- **Stop, a split button:**
  - **Stop after this track** (the main action): nothing is lost. The header says "Stopping after this track…" until it ends.
  - **Stop now** (from the arrow): first a confirm, "The current track will be discarded and recorded next time."
  - Either way, the run ends as **Stopped**. Items left in the queue aren't recorded this time. Tracked playlists are picked up by the next run, and albums stay Queued.

**Now-playing strip**
Styled like the client's now-playing bar, docked under the header:
- a small cover, title and artist;
- a track progress bar (from `track.started`'s length);
- a level meter (Spytify's `level` peaks, 10 times a second);
- a session-state pill: Waiting, Recording, Paused or Stopping.

On other pages, the same strip is the global bottom bar. It's hidden on the Dashboard, where the strip sits at the top.

**Queue**
One row per item, in run order. Row states:

| State | Row |
|---|---|
| Done | Collapsed: "✓ OK Computer · 12 recorded · 0 failed". |
| Up to date | Collapsed: "Chill Mix · nothing new". All the files already exist, so nothing was recorded. |
| Current | Expanded (see below). |
| Finishing | "Finishing OK Computer… saving the last track", with an indeterminate bar. It covers the ~25–30 s between items. Spotify pauses at the end of the item, and the recorder waits 20 s, saves the last track and exits. |
| Interrupted · will retry | Playback paused for more than 20 s in the middle of the item (a network stall, say), so the recorder ended it early. The controller puts the item back at the end of the queue, and the row moves there with this pill. Tracks already recorded are skipped as duplicates on the retry. If the same item is interrupted twice in one run, it ends as Partial. |
| Carried over | Cut off when the last window ended. It runs first. |
| Up next | Cover, name and what to expect: "84 new", "11 tracks", or "checking" for followed playlists. Its ⋯ menu has **Remove from this run**. |

**Current item, expanded inline**
- The item's track table ticks off live and auto-scrolls to the playing track. Each track is in one of these states:
  - ✓ recorded, with its size;
  - ↷ skipped (duplicate);
  - ● recording, with progress;
  - ○ pending;
  - ✕ failed, with the reason.
- For followed or linked playlists, whose track list the API won't give, the table grows as tracks start, and the header shows "n recorded" with no total.

**"Back up now" from the Library or Albums during a run:** the item goes into the queue right after the current item, with a toast: "Added to the current run".

### Warnings during a run
Inline alerts sit between the strip and the queue.

- **Playback moved to another device** (you started Spotify on your phone, for example):
  - The current track is discarded. The current item is marked **Interrupted**, and the run **pauses**. It never takes playback back on its own.
  - The alert reads "Playback moved to <device>. The backup is paused so it won't interrupt you." with **Resume** and **Stop**.
  - **Resume** takes playback back on the container's device and restarts the current item. Tracks already recorded are skipped as duplicates within seconds, so it carries on where it stopped.
  - **Scheduled runs while paused:** a paused run stays paused until you resume or stop it. The next scheduled run doesn't start while one is paused, and History notes it as skipped.
- **Spytify error:** an `error` event shows as a red alert with the message. If the current item can't continue, it is marked Partial and the run moves to the next item.
- **Track failed** (for example, a low-volume take): no alert. It shows in the track table and the counters.

### Dashboard endpoints

| Method | Path | Notes |
|---|---|---|
| GET | `/api/status` | `state` (`idle`, `running`, `paused`, `stopping` or `finishing`), the current run, item and track, and the counters. Read on first load; after that the WS keeps it current. |
| GET | `/api/schedule/next` | `{at, playlists_to_check, albums_queued, held_reason}` |
| GET | `/api/runs/last` | The last run's summary and its top issues. |
| GET | `/api/backlog` | `{tracks_total, tracks_recorded, seconds_total, seconds_left, unknown_items, finish_at, as_of}`. `finish_at` is null with Manual only. Shared with wizard step 8 and the Schedule preview. |
| POST | `/api/runs` | Run now. Without `items` it runs everything due; with `{items: [...]}` it adds those items (from the Library or Albums). |
| GET | `/api/runs/current` | The queue, with per-item state and the current item's track list. |
| POST | `/api/runs/current/stop` | `{mode: "after_track" \| "now"}` |
| POST | `/api/runs/current/resume` | After a device takeover. |
| DELETE | `/api/runs/current/items/{id}` | Removes an Up next item. |
| WS | `/api/events` | Pushes these events:<br>• run: `run.state`, `run.queue`, `item.state`<br>• `backlog`, after each item and each catalog refresh<br>• Spytify, relayed: `track.started`, `track.finished`, `track.skipped`, `level`, `session.state`<br>• warnings: `device.moved`, `error` |

---

## Schedule

Its own sidebar page. The Dashboard's Next backup card links here.

Recording happens in real time: a 120-track playlist takes about 8 hours, and a first full backup can take days. So the schedule is a **weekly set of allowed hours**:
- A scheduled run records only during active hours.
- Whatever is left carries over to the next active hours.

### The page
- **Summary:** a read-only mini version of the week grid (described below), with one line of text, for example "Mon–Fri 01:00–07:00 · Sat–Sun 00:00–09:00 · 44 h a week".
- **Button:** a primary **Edit schedule** button, which opens the schedule modal.
- **Empty grid:** "No backups scheduled · Run now only".
- **Time zone:** a read-only line: "Times are in Europe/Berlin. Set by `TZ` in the UGOS Docker app."
- **Run order and Preview:** the two sections below.

### Schedule modal: the week grid
A large modal in the client's dialog style. Its main element is a grid of squares:
- **Columns:** 24 one-hour blocks, 00:00–24:00. The header labels every third hour (00, 03, 06 …).
- **Rows:** the seven weekdays, Monday first, labelled Mon–Sun.
- **Cells:**
  - **Active:** filled in the accent green.
  - **Inactive:** the dark surface colour, with a subtle outline.
  - **Hover:** shows the cell's time ("Tue 02:00–03:00").

**Selecting hours:**
- **Click a cell** to toggle it.
- **Click and drag** to paint across cells, in any direction.
  - The first cell decides what the drag does. Starting on an inactive cell turns cells on; starting on an active cell turns them off.
  - It works with a mouse or a finger (pointer events).
- **Click a day label** to toggle that whole day.
- **Click an hour label** to toggle that hour on every day.

**Under the grid:**
- A live summary line, as on the page ("Mon–Fri 01:00–07:00 · … · 44 h a week").
- **Clear all**, at the left.
- **Cancel** and **Save**, at the right.

**Contiguous blocks form one window,** across midnight too. Mon 23:00 and Tue 00:00 are one window, 23:00–01:00.

**On a phone,** the grid keeps square cells and scrolls sideways, with the day labels pinned at the left.

### How runs follow the grid
- **Start:** a scheduled run starts at the first hour of each window, if there's anything to do.
- **End of a window:** the run **stops after the current track**, so nothing is half-recorded.
  - The cut-off item keeps what it recorded. It's marked **Carried over** and goes first in the next window, where its recorded tracks are skipped as duplicates within seconds.
  - The rest of the queue waits for the next window.
  - The run ends as **Window ended** in History, with its counts.
- **Start-up (missed runs):** if the container starts during an active hour, for example after the NAS was off, the run starts straight away and stops at the window's end as usual. A window that passed while the container was off is recorded in History as "Missed: spotibackup was not running".
- **Run now ignores the grid.** A forced run goes until it's done or you stop it. If a window opens while one is running, no second run starts; the forced run already covers it.
- **Editing during a run:** if you turn off the current hour, the run stops after the current track.

### Run order
A fixed rule, shown on the page as a short list:
1. Carried-over items.
2. Liked Songs.
3. Tracked playlists with new tracks, most recently changed first.
4. Queued and Partial albums, oldest added first.

Followed playlists, which have no track totals, are checked along with the other playlists. Items that are up to date take seconds.

### Preview
Shown on the page and in the modal, where it updates as you paint:
- **Next windows:** the next three, for example "Tue 12 Nov · 01:00–07:00".
- **Backlog:**
  - "412 tracks waiting · about 27 h of recording · about 4 days at this schedule".
  - Followed playlists can't be estimated, so it adds "+ followed playlists (unknown)".
  - When nothing is waiting: "Everything is up to date".
  - It's the same [backlog estimate](design.md#backlog-estimate) as the Dashboard's Full backup card, worked out against the grid being edited.

### Schedule endpoints

| Method | Path | Notes |
|---|---|---|
| GET | `/api/schedule` | `{hours: {mon: [1,2,3,4,5,6], tue: […], …}, tz}`. Each list holds the active hours of that day (0–23). All lists empty means Manual only. |
| PUT | `/api/schedule` | Saves. Same shape. |
| POST | `/api/schedule/preview` | Body: `{hours}`, which may be unsaved. Returns `{windows_per_week, hours_per_week, next_windows: [...], backlog}`. `backlog` has the same shape as `GET /api/backlog`, with `finish_at` worked out for the posted hours. |
| GET | `/api/schedule/next` | Shared with the Dashboard: `{at, window_end, playlists_to_check, albums_queued, held_reason}`. |

---

## History

What was backed up, when, and what went wrong. It has three tabs, styled like the client's filter chips under the page title: **Runs**, **Tracks** and **Logs**.

History is kept **forever**. The rows are tiny, and the per-track records are also how the Library and Albums know what's recorded. Logs are the exception: they rotate (see the Logs tab).

### Runs tab: the run list
A table, newest first, loading more as you scroll.

| Column | Content |
|---|---|
| When | "Tue 12 Nov 01:00" |
| Trigger | Scheduled · Run now · Start-up (a run started at start-up, during an active hour) |
| Duration | "5 h 12 m" |
| Outcome | Pill: Completed · Completed with issues · Window ended · Stopped · Failed |
| ✓ | Tracks recorded |
| ↷ | Tracks skipped as duplicates |
| ✕ | Tracks failed |
| Size | Space added: "+612 MB" |

- **Rows with no run:** greyed, single-line rows record runs that didn't happen, with no counts:
  - "Missed: spotibackup was not running";
  - "Skipped: the previous run was paused";
  - "Held: Spotify connection expired".
- **Filter chips:** All · With issues · Scheduled · Run now.
- **Opening a run:** clicking a row opens the run page.

Duplicate skips are cheap. In the prototype, a re-run of an already-recorded album skipped every track in under a second. So a large ↷ count with a small ✓ count is the normal look of a nightly run, and it isn't treated as a problem.

### Run page
- **Header:**
  - Date, trigger, start–end time and duration.
  - The window it ran in, for scheduled runs.
  - The outcome pill, the ✓ / ↷ / ✕ counts and the size added.
  - A link, "Logs for this run", which opens the Logs tab filtered to this run.
- **Items:** in the order they ran, each one a collapsible row:
  - The cover, name, and ✓ / ↷ / ✕ counts.
  - **Badges where they apply:**
    - Up to date: nothing new;
    - Interrupted, retried;
    - Interrupted twice: Partial;
    - Carried over from the previous run;
    - Carried over to the next run;
    - Removed from run;
    - Stopped.
  - **Expanded:** the item's tracks for this run. The columns are #, title, artist and result:
    - **Recorded:** the size, and the file path with a copy button.
    - **Skipped (duplicate).**
    - **Skipped (ad):** Spytify's other skip reasons are shown the same way.
    - **Failed:** the reason, such as "low volume", "recorder error", "track unavailable", "interrupted" or "discarded by Stop now".
- **Actions:**
  - **Retry failed** on an item with failures: adds that item to a forced run. Only the missing tracks get recorded; the rest are skipped as duplicates.
  - **Retry all failed** in the header: does the same for every such item.

### Tracks tab
Every track spotibackup has ever handled, one row each.
- **Search:** by title, artist or album.
- **Filter chips:** All · Recorded · Failed (not recorded yet). The Failed filter is the to-do list.
- **Columns:**
  - title and artist;
  - album;
  - "In": the playlists and albums the track belongs to;
  - latest result;
  - recorded at;
  - size.
- **Clicking a row** opens a side panel:
  - The file path, with a copy button.
  - Each attempt across runs: the run, the result and the reason. Each run links to its run page.
  - **Retry**, for failed tracks: queues the item the track belongs to.

### Logs tab
The raw output, for debugging.
- **Live tail:**
  - New lines stream in, and the view follows the bottom.
  - Scrolling up pauses the follow, and a "Jump to latest" pill appears.
- **Filters:**
  - **Source:** chips for Controller · Spytify · Spotify client · System (s6 services).
  - **Level:** chips for Info · Warning · Error.
  - **Run:** a dropdown with "All" or a single run. "Logs for this run" on a run page sets it.
  - **Text search.**
- **Line format:** a monospace line with the time, a source badge, a level colour and the message.
- **Download:** saves the last 7 days of logs as a `.zip`, for bug reports.
- **Retention:** logs rotate after 14 days or 50 MB, whichever comes first. The tab says so in its footer.

### History endpoints

| Method | Path | Notes |
|---|---|---|
| GET | `/api/runs` | Takes `?cursor=&filter=`. Returns a page of runs, including the missed, skipped and held rows. |
| GET | `/api/runs/{id}` | The run header and its items, with counts and badges. |
| GET | `/api/runs/{id}/items/{item}/tracks` | The item's tracks and their results in that run. |
| POST | `/api/runs` | `{items: [...]}`. Retry failed or retry all failed. Shared with the Dashboard. |
| GET | `/api/tracks` | Takes `?q=&result=&cursor=`. |
| GET | `/api/tracks/{id}` | The track's file and its attempts. |
| GET | `/api/logs` | Takes `?source=&level=&run=&q=&before=`. Returns a page of older lines. |
| WS | `/api/events` | `log.line` for the live tail, and `run.finished` to refresh the list. |
| GET | `/api/logs/download?days=7` | The `.zip`. |

---

## Settings

One scrolling page, like the client's Settings: section headings, with rows of label, description and control.
- **Saving:** changes collect in a sticky "Unsaved changes · Discard · Save" bar at the bottom.
- **Mid-run changes:** the recorder only runs during a backup, and receives its settings when each item starts. So a change saved during a run applies from the next item.

### Recording
- **Format:** AAC 256 (default) · AAC 320 · MP3 320.
  - "More formats" expands to FLAC and WAV, each marked "No benefit: the source is lossy (Ogg Vorbis 320). Bigger files, same quality."
  - A note under it: "Applies to new recordings. Tracks already recorded aren't recorded again."

### Files
- **Layout:** fixed, with no setting. Each track is saved once, as `Music/<artist>/<album>/<artist> - <title>.m4a`. The row shows that path as an example.
- **Playlist files:** a toggle, "Write .m3u playlists", on by default.
  - Each tracked playlist, and Liked Songs, gets `Music/Playlists/<name>.m3u`, in Spotify's order, with paths relative to the music folder.
  - Plex, Jellyfin and Navidrome read these.
- **Music folder:** read-only. It shows `/music → /volume6/Music`, the free space, and a "Check access" button that re-runs the wizard's step-6 check.

### Library
- **Automatically track playlists I create:** a toggle. It's the same setting as in the Library's ⋯ menu and wizard step 7.

### Notifications
Two channels share one event table.

| Event | Browser | Webhook |
|---|---|---|
| Run finished with issues | ☑ | ☑ |
| Run finished, everything fine | ☐ | ☐ |
| Run paused: playback moved to another device | ☑ | ☑ |
| Run failed or held (connection expired, client logged out, setting drifted) | ☑ | ☑ |
| Spotify connection expires in 4 weeks / 1 week | ☑ | ☑ |
| Music folder low on space (under 5 GB) | ☑ | ☑ |

**Browser notifications:**
- An **Enable** button asks the browser for permission. The row then shows the state: Enabled, Blocked by the browser (with how to unblock), or Unavailable (see below).
- They are delivered while any spotibackup tab is open, even in the background, from the live event stream.
- **Plain HTTP:** browsers only allow notifications on secure pages (HTTPS, or localhost). Opened over plain `http://<nas>:8080`, the row shows **Unavailable** and explains how to reach spotibackup over HTTPS, for example through the UGOS reverse proxy. See the open questions.

**Webhook:**
- **URL field**, plus an optional secret. The secret is sent as an `X-Spotibackup-Secret` header.
- **Send test** posts a sample event.
- **Payload:** `POST` JSON: `{event, at, title, message, run_id?, url}`. `url` deep-links into the UI.
- **Uses:** Home Assistant, n8n, ntfy (which accepts a JSON POST) and the like.

### Security
- **Password for this web UI:** off by default, for LAN-only use.
  - **Set password** opens a dialog with new password and confirm. After that, the UI shows a simple sign-in screen, and the session lasts 30 days per browser.
  - **Change** and **Remove** need the current password.
- **Env var:** if `UI_PASSWORD` is set in the UGOS Docker app, this row is read-only: "Set by the Docker app (`UI_PASSWORD`)".
- **Storage:** only a hash of the password is stored, in `/config`.

### Spotify account (Web API)
- **Connection:** "Connected as Lorne · authorized 12 Nov 2026 · expires 12 May 2027", with a status pill: OK, Expiring or Expired.
- **Reconnect:** opens wizard step 5 on its own.
- **Client ID:** masked, with a **Change** button. Changing it disconnects and reconnects.
- **Disconnect:** deletes the token. Scheduled runs are held until you reconnect.

### Spotify client
- **Client:** its version, and its login state ("Logged in", detected by MPRIS).
- **Settings checklist:** the same as wizard step 4, with each row's pill and a **Re-check** button. It links to Player to change anything.
- **Open Player:** opens the full-size noVNC view.

### Container
Read-only, with the note "Change these in the UGOS Docker app":
- **Device name** (`DEVICE_NAME`, default `spotibackup`), with the warning "Changing it may log the Spotify client out; you'd scan the QR code again".
- **Time zone** (`TZ`).
- **Music mapping** (`/music`).
- **`UMASK`.**
- **Versions:** the image, Spytify and the Spotify client.

### About
- **Version** and a link to the project on GitHub.
- **Terms notice:** "Read the Terms notice again" reopens the wizard's step-1 text, read-only.
- **Run setup again:** reopens the wizard from step 2.

### Settings endpoints

| Method | Path | Notes |
|---|---|---|
| GET/PUT | `/api/settings` | Body:<br>• `format`, `bitrate`, `write_m3u`, `auto_track_owned`<br>• `notifications: {browser: [events], webhook: {url, secret, events}}` |
| POST | `/api/notifications/test` | `{channel: "webhook"}` |
| GET | `/api/storage` | Shared with wizard step 6. |
| GET | `/api/auth` | `{password_set, set_by_env}` |
| PUT | `/api/auth/password` | `{current?, new}` sets or changes the password. |
| DELETE | `/api/auth/password` | `{current}` removes the password. |
| POST | `/api/auth/login`, `/api/auth/logout` | Session cookie. |
| GET | `/api/spotify/connection` | Shared with the banner and wizard step 5. |
| DELETE | `/api/spotify/connection` | Disconnect. |
| GET | `/api/client/status`, `/api/client/checklist` | Shared with the wizard. |
| GET | `/api/container` | Read-only env values and versions. |

---

## Player

The Spotify client's screen at full size, through noVNC. Use it for anything the wizard doesn't cover: changing a client setting, checking what the client shows, logging in again. Wizard steps 3 and 4 embed the same view component at a smaller size.

### Toolbar
A slim bar across the top, with icon buttons and tooltips:
- **Scale:** a two-way toggle.
  - **Fit** (default): the virtual screen is scaled to the window.
  - **100%:** pixel-exact, and scrolls if the screen is larger than the window.
- **Fullscreen:** the browser's fullscreen for the view. Esc leaves it.
- **Restart client:** asks first: "Restart the Spotify client? It comes back logged in."
  - It's disabled during a run, with the tooltip "Not available while a backup is recording".
  - It's also how you get a fresh login QR code.

### Status strip
One line under the toolbar:
- the client version;
- "Logged in" (detected by MPRIS) or "Not logged in", which links to wizard step 3;
- the VNC connection state: Connected, Reconnecting… or Disconnected, the last with a Retry button.

### During a run: view-only
While a backup is recording:
- **View-only:** the view ignores the mouse and keyboard, so a stray click can't pause or skip what's being recorded. The view dims slightly and shows a lock icon in the corner.
- **Banner:** above the view, a yellow banner reads "A backup is recording. The view is read-only so a click doesn't spoil it." with a **Take control** button.
- **Take control:** asks first: "Clicks in the client can pause or skip the track being recorded. Take control anyway?" Confirming unlocks input. The run carries on; nothing is paused for you.
- **When the run ends,** the view goes back to interactive. If a run starts while you have control, the view locks again and the banner returns.

View-only is set on the noVNC client, which is a guard against mistakes, not a security boundary. Access to the UI as a whole is protected by the optional password.

### Player endpoints

| Method | Path | Notes |
|---|---|---|
| WS | `/vnc` | The noVNC websockify proxy. |
| GET | `/api/client/status` | Version and login state. Shared with the wizard and Settings. |
| POST | `/api/client/restart` | Refused with `409` during a run. |
| GET | `/api/status` | Whether a run is active, which decides view-only. The WS `run.state` event keeps it current. |

---

## User flows

1. **First-time setup:** the Setup wizard, steps 1–8, ending in the Library or a first run.
2. **Picking playlists:**
   1. In the Library, turn tracking on or off per row, or by the grid badge.
   2. Press **+**, then add followed playlists from the suggestions, or paste a link.
   3. Tracked items are picked up by the next scheduled run, or right away with "Back up now" on a row, in the detail view, or in the options menu.
   4. **Albums:** open Albums and press **+**. Filter by name, then Add. Added albums are queued for the next run. After a complete recording, they're done.
3. **Scheduled run:**
   1. At the first hour of a window in the schedule grid (or on start-up during an active hour), the Dashboard switches from idle to running.
   2. Items run in the fixed order, each followed by its Finishing state.
   3. If the queue empties first, the run ends as Completed.
   4. At the end of the window (the last active hour), the run stops after the current track. The cut-off item is Carried over, and the run ends as Window ended.
   5. The Dashboard shows idle again, with the Last run card filled in and the next window.
4. **Forced run:**
   - **Run now** on the Dashboard runs everything due.
   - **Back up now** or **Record now** on an item (Library, Albums or a detail view) runs just that item.
   - If a run is already going, the item goes in right after the current item.
   - A forced run ignores the schedule window.
5. **Failure:** to be written with the Dashboard and History pages. Known cases so far:
   - The Web API token has expired (`invalid_grant`): the run is held, and the banner offers Reconnect.
   - A client setting has drifted (an Auto row fails the pre-run check): the run doesn't start, and the message links to the checklist.
   - Missed window (the container was off and the window is over): the run is skipped, and History says why.
   - Playback moved to another device: the item is Interrupted, and the run pauses until you press Resume or Stop.
   - Playback paused for more than 20 s for another reason: the item is Interrupted and retried once at the end of the queue. If it's interrupted again, it ends as Partial.
   - A Spytify error: the item is marked Partial, and the run goes on to the next item.

## Open questions

The feasibility prototype (`prototype/`, on a parallel branch) settles these. The UI above takes the more cautious answer until then.

1. ~~**Login in Xvfb:** does the client's login work inside Xvfb through noVNC?~~ **Resolved:** yes, by QR code scanned with the phone app. No browser hand-off. See step 3.
2. **Detecting login:** MPRIS works in the container, but which signal reliably means "logged in"? Candidates: MPRIS metadata appearing, the client's user folder in `/config`, or the Web API device list.
3. **Reading client settings:** Spytify's `core.info` reports crossfade and gapless. Can the controller also read autoplay and streaming quality from the client's prefs file in `/config`? If not, those two rows are manual checkboxes.
4. ~~**OAuth redirect:**~~ **Resolved:** PKCE with `http://127.0.0.1:8888/callback`. You paste the error page's address back. Refresh tokens expire 6 months after you authorize, so there's a Reconnect flow.
5. ~~**Container client as a device:**~~ **Resolved:** yes. It's a Connect "Computer" named after the container's hostname, set by the `DEVICE_NAME` env var (default `spotibackup`).
6. ~~**Lossless:**~~ **Resolved:** not available. The client tops out at Ogg Vorbis 320, so FLAC and WAV are marked "no benefit".
7. ~~**Playlists made by Spotify:**~~ **Decided:** not supported. There is no link input.
8. ~~**Client login surviving a recreate:**~~ **Resolved:** the login survives. The prototype recreated the container with `--force-recreate`, and the device kept its name and ID. This needs a stable `/config/machine-id` and an unchanged `DEVICE_NAME`; the container's real hostname doesn't matter. Image updates don't need a new QR scan. Changing `DEVICE_NAME` is untested, so Settings keeps its warning. A "client logged out" Dashboard state is still useful as a fallback.
9. **Liked Songs playback:** can `PUT /me/player/play` start Liked Songs as a context, or must the controller pass its track URIs in batches? Either way, the end of the item has to be detected the same way.
10. **Linked playlists:** for a public playlist you don't follow, does `GET /playlists/{id}` return name and cover (for the modal's preview) when `/items` returns 403? And does playback work, as it does for followed playlists? **Documented, not tested:** Spotify's February 2026 migration guide says such playlists return metadata without `items`. Playback still needs a live test.
11. **Saved albums access:** do the saved-album endpoints (`/me/albums`, and album tracks) work for a 2026 dev-mode app? The Albums page depends on them. **Documented, not tested:** the February 2026 changelog lists both as still available, and `/me/albums` still includes each album's tracks with `duration_ms`. Needs a live test.
12. **Browser notifications over HTTP:** notifications need a secure context. Can the image serve HTTPS itself (for example, with a self-signed certificate on a second port)? Or should the docs point to the UGOS reverse proxy? With a self-signed certificate the browser shows a warning, and Chrome may still refuse notifications.
13. **Client install timing:** is installing the client on first start (not baked into the image) fast and reliable enough to show as a wizard step?
