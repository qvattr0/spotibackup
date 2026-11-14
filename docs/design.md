# spotibackup design

spotibackup is a single Docker image for a UGREEN NASync DXP4800+ (x86_64, UGOS Pro). It backs up your Spotify playlists, and optionally your saved albums, by recording them with [Spytify 2.0](https://github.com/spytify/spytify). A built-in web UI ([web-ui.md](web-ui.md)) controls it.

## Goals and non-goals

**Goals:**
- One image, installed through the UGOS Pro Docker app, with its ports, volumes and environment editable there.
- A web UI to choose what to back up, schedule or force runs, watch progress and set the format.
- Files in a NAS shared folder, with permissions left to that folder.

**Non-goals:** ARM hosts, more than one Spotify account.

## Feasibility summary

**Spytify:** `spytifyd` speaks newline-delimited JSON-RPC 2.0 on a Unix socket. The controller runs it once per item as `spytifyd --once --serve --idle-exit 20s`.
- **Actions:** `session.start`, `session.stop` and `session.status`; `settings.get` and `settings.set`; `player.control`; `log.tail`; `core.info`.
- **Pushed events:** `session.state`, `track.started`, `track.finished` (with the file path), `track.skipped` (duplicate, ad and so on), `level` and `error`.

**The Spotify client:**
- MPRIS over D-Bus gives local now-playing.
- Through the Web API it is a Connect device: playlists come from `GET /me/playlists`, and playback is started with `PUT /me/player/play`.

**One-time GUI steps:** done through noVNC: logging in, turning off crossfade, gapless playback and autoplay, and setting streaming quality.

**Recording order:** Spytify never records a track that was already playing when the session started. The controller starts the session before playback.

**Formats:** AAC (`.m4a`), MP3, FLAC or WAV, at 128, 160, 256 or 320 kbps. The default is AAC 256. The prototype found that the Linux client (1.3.4) streams at most Ogg Vorbis 320 kbps, with no Lossless, so FLAC and WAV add size without quality.

**Login:**
- **Client:** logs in by QR code scanned with the Spotify phone app, shown through noVNC. No browser is needed in the image.
- **Web API:** uses PKCE with the redirect `http://127.0.0.1:8888/callback`, and you paste the resulting address back. Refresh tokens expire 6 months after you authorize (Spotify, 2026-06-18), so the UI tracks the expiry and offers Reconnect.

**Base image:** Spotify Linux 1.2.77+ needs glibc 2.38+, so the image is Ubuntu 24.04 or Debian 13.

**Developer app:** since March 2026, a Development Mode app needs a Premium owner and allows up to five users. The same change narrowed the Web API: see [Web API usage](#web-api-usage).

## Container

One container, with s6-overlay supervising:

| Service | Role |
|---|---|
| Xvfb + window manager | The virtual display the client runs on. |
| x11vnc + noVNC | Views that display in the browser, proxied by the controller at `/vnc`. |
| D-Bus session | MPRIS and the client's IPC. |
| PulseAudio | A null sink at 44.1 kHz. Spytify records from it. |
| Spotify client | Installed from Spotify's apt repo on first start into `/config`, not baked into the public image. |
| `spytifyd` | The recorder. The controller starts it for each item (`--once --serve --idle-exit 20s`), so it is not a long-running service, and the controller passes the recorder settings in at the start. |
| controller | Web UI and REST/WS API, run engine, scheduler, SQLite in `/config`. |

**UGOS configuration:**
- `EXPOSE 8080`.
- `VOLUME /config /music`: map `/music` to `/volume6/Music`.
- `ENV`:
  - `TZ`
  - `UMASK=000`
  - `UI_PASSWORD` (optional): requires a password for the web UI. It can also be set in Settings.
  - `DEVICE_NAME` (default `spotibackup`): the name of the Spotify Connect device ("Playing on …"). A small preloaded library sets it, because the client takes its name from `gethostname()`. Recreating the container (for example, for an image update) keeps the login: tested with `--force-recreate`. That relies on a stable `/config/machine-id` and an unchanged `DEVICE_NAME`. Changing `DEVICE_NAME` is untested.
- A `docker-compose.yml` for the Docker app's Project feature.
- Avoids host ports 9999 and 9443, which UGOS uses.

**Spytify defaults:**
- `privacy.bug_tracker=off`
- `recorder.duplicates=skip`
- `recorder.skip_duplicates_forward=on`
- `output.group_by=artist/album`

## Run flow, per playlist or album

1. Fetch the track list through the Web API. Skip the item if every file already exists. A playlist without item access (see [Web API usage](#web-api-usage)) has no track list, so it is always played.
2. Pause the player and turn shuffle and repeat off.
3. Start `spytifyd` for this item with the current recorder settings, then call `session.start`.
4. `PUT /me/player/play` with the container's `device_id` and the item's `context_uri`.
5. Follow Spytify's events. Spotify pauses at the end of an album or playlist. After 20 s of silence, `spytifyd` saves the last track and exits (`--idle-exit 20s`), about 25–30 s after the pause.
   - **No client restart is needed.**
   - **A pause of more than 20 s in the middle of an item** ends it early. This happens when another device takes over or the network stalls. The controller re-queues the item, and the tracks already recorded are skipped as duplicates.
6. Write a `.m3u` for the playlist, in the Web API's track order. For a playlist without item access, use the order the tracks were recorded in.

The scheduler queues items. An item forced from the UI goes right after the current item. One item records at a time.

## Web API usage

Spotify narrowed the Web API for Development Mode apps in February 2026: new apps from 11 February, existing apps from 9 March. spotibackup only uses endpoints that are still available ([migration guide](https://developer.spotify.com/documentation/web-api/tutorials/february-2026-migration-guide)).

| Endpoint | Used for | Page size |
|---|---|---|
| `GET /me` | The display name ("Connected as …"). | — |
| `GET /me/playlists` | The playlist list, each with its owner, `snapshot_id` and `items.total`. | 50 |
| `GET /playlists/{id}/items` | A playlist's tracks. It replaces the removed `/playlists/{id}/tracks`. | 50 |
| `GET /playlists/{id}` | Name, owner and cover for a pasted link. | — |
| `GET /me/tracks` | Liked Songs, newest first. | 50 |
| `GET /me/albums` | Saved albums, each with its first page of tracks. | 50 |
| `GET /albums/{id}/tracks` | The rest of an album longer than that first page. | 50 |
| `GET /me/player/devices` and the player controls (`play`, `pause`, `shuffle`, `repeat`) | Finding the container's device and starting each item. | — |

**The 2026 rules that shape the design:**
- **Item access:** `/items` returns tracks only for playlists you own or collaborate on. Any other playlist returns 403, and `/me/playlists` leaves out its `items` total. So followed and linked playlists have no track list or total until they're recorded. A followed playlist you collaborate on has access and behaves like your own.
- **Renamed fields:** a playlist's contents moved from `tracks` to `items`, and each entry's track from `track` to `item`. The controller reads only the new names.
- **No batch lookups:** `GET /tracks?ids=` and `GET /albums?ids=` are gone. Everything the controller needs, `duration_ms` included, comes with the listings above, so it's stored when it's fetched.
- **Removed fields:** among others, `popularity` and `linked_from` on tracks, and `label` on albums. spotibackup uses none of them.
- **Rate limits:** Spotify publishes no Development Mode numbers. Limits apply over a rolling 30 s window, plus quota buckets shared by every app on the developer account. A `429` carries `Retry-After`; see Requests below for how the controller handles it.

**Load:**
- Each request names only the fields it needs through the `fields` parameter: track ID, title, artists, album, cover, `duration_ms` and `is_local`.
- **Playlists:** the controller stores each one's `snapshot_id` and refetches its items only when that changes.
- **Liked Songs:** it has no `snapshot_id`. The controller reads from the newest page until it reaches a track it already has, then compares `total`. If the total doesn't match, it refetches the whole list.
- **Albums:** they don't change, so they're fetched once, when added.
- **Example:** a first fetch of 23 playlists with 3,000 tracks, 2,000 Liked Songs and 100 saved albums is about 110 requests. A refresh where nothing changed is a few.

**Requests:**
- **One at a time:** the controller sends one Web API request at a time, with a short pause between them (250 ms by default) during catalog refreshes. Playback calls during a run aren't delayed.
- **Retries:**
  - A `429` waits for `Retry-After`.
  - A `5xx`, a `503` or a timeout is retried after 5 s, and the delay doubles on each further attempt, up to 3 retries. If a `429` also carries `Retry-After`, the longer of the two wins.
  - A `401` refreshes the token once, then retries. A `403` or `404` is not retried: it's an answer (no item access, or the item is gone).
- **Counting:** the controller counts requests per catalog refresh and per run, with retries and `429`s counted separately. Each refresh and each run ends with one Controller log line, for example "Web API: 112 requests · 2 retried · 0 rate-limited". A rising rate-limited count is the early sign of a quota problem.

**Not used:** some tools, such as [zotify](https://github.com/Googolplexed0/zotify), log in as Spotify's own desktop client and call Spotify's internal endpoints. That gets around every Development Mode limit, followed playlists included. spotibackup doesn't do this:
- it passes the controller off as Spotify's own client, which breaks the developer terms more directly than recording does;
- it raises the risk to an account that's used every night;
- the internal endpoints are undocumented and change without notice;
- it saves few requests here, because Liked Songs and albums cost the same either way.

## Backlog estimate

Recording happens in real time, so a backup takes about as long as the music still to record. The Dashboard's Full backup card, wizard step 8 and the Schedule preview all show this estimate, from one calculation (`GET /api/backlog`).

- **Catalog:** every tracked item's track list with `duration_ms`, kept in SQLite and refreshed as described in [Web API usage](#web-api-usage). It's refreshed at the start of each run, and when the Dashboard or Schedule page opens, at most once every 15 minutes.
- **What's recorded:** History's per-track records, keyed by Spotify track ID.
- **Time left:**
  - the sum of `duration_ms` over tracks not recorded yet, counting each track ID once however many items it's in;
  - plus about 35 s for each item with something to record: a few seconds to start, and the 25–30 s Finishing tail;
  - local files (`is_local`) and entries with no `item` are left out, because they can't be played.
- **Finish date:** the controller walks the schedule's active hours forward from now until the time left is used up. With Manual only, there's no date.
- **Unknown items:** followed and linked playlists have no track list, so they're left out of the figure and counted separately.
- **Accuracy:** duplicates are skipped in seconds, so they barely count. The same song can have a different track ID on a single and on its album; it's counted twice but recorded once, so the estimate leans slightly high.

## Distribution

GitHub Actions builds an amd64 image and pushes it to Docker Hub.

## Terms of Service

This project breaks Spotify's terms:
- **User terms:** Spotify's user guidelines prohibit ripping and recording content.
- **Developer terms:** Spotify's developer terms forbid facilitating stream ripping.
- **Precedent:** accounts using recording software have been suspended before.

**Practical risks:**
- The Premium account is suspended or closed.
- The developer app is revoked.
- Nightly automated plays and skips look like artificial streaming.

Whether private copies are legal depends on the country, and this document doesn't cover it. The setup wizard shows this notice once. Using a separate Premium account lowers the risk to your main one.

## Risks to validate

Open questions are also tracked in [web-ui.md § Open questions](web-ui.md#open-questions).

The prototype settled these (details in `prototype/FINDINGS.md`):
- Login inside Xvfb works, by QR code.
- The client has no Lossless.
- The container is a Connect device.
- The end of an item is handled by Spytify's `--idle-exit 20s`: the last track is saved about 27 s after the pause.
- Recording works end to end.

Still to validate:
- **Saved albums:** Spotify's February 2026 changelog lists `/me/albums` and `/albums/{id}/tracks` as still available. Needs a live test.
- **Followed playlists:** Spotify documents the 403 on `/items`. Whether they still play by `context_uri` needs a live test.
- **Field names:** Spotify's reference pages still show some fields the changelog removed, so the fields the controller reads need checking against a live response.

## Component Rules

Rules for the components in `ui-design/spotibackup.pen`. Follow them when editing the design file.

### Shapes vs. icons in a leading slot

Some components take either a **shape** or an **icon** in the same slot. A Lucide icon has built-in whitespace inside its box, while a shape doesn't. So the padding on an icon's side is 2 px smaller than on a shape's side, which makes both look the same distance from the edge.

### Status pill (`Pill` component)

| Leading element | Padding (top, right, bottom, left) |
| --- | --- |
| Shape: 6 px `Dot` ellipse (the component's default) | `3, 8, 3, 8` |
| Icon: 12 px Lucide icon, for example the dashed `circle-dot-dashed` for "Recording" | `3, 8, 3, 6` |

- When you swap a pill's dot for an icon, also override that instance's padding to `3, 8, 3, 6`. When you swap it back to a dot, remove the override.
