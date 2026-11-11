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

**Developer app:** since March 2026, a Development Mode app needs a Premium owner and allows up to five users.

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

1. Fetch the track list through the Web API. Skip the item if every file already exists.
2. Pause the player and turn shuffle and repeat off.
3. Start `spytifyd` for this item with the current recorder settings, then call `session.start`.
4. `PUT /me/player/play` with the container's `device_id` and the item's `context_uri`.
5. Follow Spytify's events. Spotify pauses at the end of an album or playlist. After 20 s of silence, `spytifyd` saves the last track and exits (`--idle-exit 20s`), about 25–30 s after the pause.
   - **No client restart is needed.**
   - **A pause of more than 20 s in the middle of an item** ends it early. This happens when another device takes over or the network stalls. The controller re-queues the item, and the tracks already recorded are skipped as duplicates.
6. Write a `.m3u` for the playlist, in the Web API's track order.

The scheduler queues items. An item forced from the UI goes right after the current item. One item records at a time.

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
- The saved-album endpoints under the 2026 dev-mode rules.
- Followed playlists return 403 on their item list, but play.
