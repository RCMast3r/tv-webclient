# nebtv web remote

A **single-file** browser dashboard for the TV. It has two control paths:

- **nebtv backend** (default) — a small service running on the TV at
  `http://samsung.lan:8099`. No pairing, no token, no Samsung API v2.
- **Samsung API v2** (fallback) — the TV's native WebSocket at
  `samsung.lan:8001`. Requires a token/pairing and does not pair on this
  firmware.

The backend path is the working one: app launch, D-pad/arrow navigation,
keyboard, text, volume and mute.

## Host the dashboard

Copy `index.html` anywhere on the LAN and serve it over plain HTTP:

```console
cd ~/my-tv/webclient
python3 -m http.server 8080 --bind 0.0.0.0
```

Open `http://<this-machine>:8080/` on desktop or iPhone.

> Serve over HTTP, not HTTPS. If the dashboard is HTTPS, the browser blocks
> `http://samsung.lan:8099` as mixed content.

`samsung.lan` must resolve to the TV. Verified on the host:
`getent hosts samsung.lan` → `192.168.86.191`.

## Backend mode

On load the dashboard calls `http://samsung.lan:8099/api/status`. If the
backend is up, the status pill shows **`samsung.lan · backend`** and the app
tiles show **backend**.

The on-TV backend (`backend/nebtv-backend.c`) uses:

| Function | Mechanism |
| --- | --- |
| Launch apps | `/usr/bin/app_launcher -s <appId>` |
| Keys / text | `/dev/uinput` virtual keyboard |
| Volume / mute | `pactl` on the TV's PulseAudio sink |

API endpoints: see `backend/README.md`.

## Controls

### D-pad and keyboard

The on-screen **D-pad** and the page keyboard capture both send TV remote keys:

| Input | Action |
| --- | --- |
| D-pad arrows / keyboard arrows | navigate |
| OK / Enter | select |
| Back | Back (sent as Escape; this TV ignores Linux `KEY_BACK`) |
| Home | Home |
| Volume / Mute | PulseAudio via the backend |
| Channel +/− | channel up/down |
| Colours | red / green / yellow / blue |

Turn on **Keyboard capture** to use the keyboard. `Ctrl`+`Shift`+`K` releases
it. The page also maps Page Up/Down, `+`/`−`, `M`, `I`, `S`, `G` and digits.

### Text

Turn on **Text input**, type in the box, press Enter. The backend types the
string as uinput key events (ASCII: letters, digits, common punctuation).

### Apps

The quick tiles launch **YouTube**, **Jellyfin**, and the custom **nebtv
Dashboard** (`9TYrsHWbaB.klyq`). The app grid comes from `/api/apps` and also
includes Netflix, Internet and Spotify.

## Samsung mode (fallback)

Open **Settings → Control backend → Samsung API v2 WebSocket**. This uses the
TV's native channel on 8001. It needs a valid pairing token. This firmware's
`msf-server` does not issue one for a fresh local client: it sends
`application.aclPairing` with null device fields and `remote-server` answers
`-4001 Unauthorized error`, so no allow prompt appears. A token from another
working client can be pasted in **Settings → Pairing token**.

The popup you may see from `org.tizen.register-device-ux` is the separate
RDM/screen-sharing ACL, not the API v2 prompt.

## Settings

| Setting | Meaning |
| --- | --- |
| Control backend | `nebtv backend` (default) or `Samsung API v2 WebSocket` |
| Backend URL | default `http://samsung.lan:8099` |
| TV hostname/IP / port | Samsung API v2 target |
| Device name | API v2 name, keep the `[Mobile]`/`[PC]` prefix |
| Pairing token | optional Samsung API v2 token |

## Troubleshooting

| Symptom | Check |
| --- | --- |
| `backend offline` | TV on, backend service running, port 8099 reachable |
| Backend service not running | `systemd-run --unit=nebtv-backend ...` or reboot; see `backend/README.md` |
| App tile does nothing | Check `/tmp/nebtv-backend.log`; verify the app ID |
| Back doesn't work | Backend maps Back to Escape; update the backend if you have an older binary |
| Volume changes but no TV OSD | Expected; volume is applied via PulseAudio |
| Keyboard letters don't type | Use **Text input**; keyboard capture is for navigation keys |

## Security

- The backend API has **no authentication**. It is LAN-only; do not expose
  port 8099 to the WAN.
- The service runs as the TV's `owner` user and uses `/dev/uinput`.
- Samsung-mode tokens are stored in `localStorage`; use **Forget token** to
  remove them.
