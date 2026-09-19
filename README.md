# Xiaomi cameras in Home Assistant through go2rtc

**English** | [Русский](README.ru.md)

A practical guide to connecting Xiaomi/Mi Home cameras to Home Assistant,
building a mobile-friendly live viewer, starting streams automatically, adding
focal-point pinch-to-zoom, and diagnosing the failures encountered in a real
deployment.

Last reviewed: **September 19, 2026**.

> go2rtc and its Xiaomi support are community projects. Not every camera model
> is supported. Never publish a Xiaomi login, token, device ID, camera IP/MAC,
> room name, private stream URL, or frame from a real home.

## 1. Resulting architecture

```text
Xiaomi camera
  │ local media; Internet used to obtain session keys
  ▼
go2rtc 1.9.14 Home Assistant add-on
  │ authenticated Home Assistant Ingress
  ▼
one active VideoRTC player per dashboard card
  ├── muted autoplay
  ├── camera selector
  ├── fullscreen
  ├── focal-point pinch-to-zoom and pan
  └── stream cleanup when the view closes
```

The validated installation played ten camera streams from several Xiaomi model
families through the external Home Assistant HTTPS route. Two disconnected
cameras returning `no-p2pid` were removed from the UI selector only. Recording,
archive access, and detection were deliberately left disabled.

## 2. Xiaomi source limitations

Xiaomi support was added in go2rtc 1.9.13. Upstream documents that:

- modern `xiaomi/miss` cameras and some `xiaomi/legacy` cameras are supported;
- each new connection needs Internet access to obtain encryption keys;
- the media connection to the camera is local;
- `miss` cameras using the `cs2` P2P protocol generally work best;
- older `legacy`/`tutk` cameras may be unreliable;
- multiple accounts, regions, dual lenses, and two-way audio are supported, but
  actual compatibility remains model-specific.

Pilot one camera from every model family before building a large dashboard.

## 3. Prerequisites

- Home Assistant OS or Supervised for the add-on and Supervisor Ingress route;
- a current Home Assistant backup;
- HACS only if additional dashboard cards are used;
- a Xiaomi/Mi Home account and access to its verification method;
- cameras and Home Assistant on the same LAN;
- a working external Home Assistant HTTPS route for remote viewing;
- enough CPU if a browser-incompatible H.265 stream must be transcoded.

For Home Assistant Container/Core, run go2rtc separately and use a protected
reverse proxy. The Supervisor Ingress API described below is not available
there.

## 4. Install the go2rtc add-on

1. Open **Settings → Apps → App store → ⋮ → Repositories**. Older Home
   Assistant versions call this Add-ons.
2. Add:

   ```text
   https://github.com/AlexxIT/hassio-addons
   ```

3. Install **go2rtc**.
4. Enable start on boot and start the add-on.
5. Open its Web UI through Ingress.
6. Review the log. Version 1.9.14 normally exposes API `:1984`, RTSP `:8554`,
   and WebRTC `:8555` inside the selected network topology.

Do not expose port `1984` to the Internet. Ingress is enough for Xiaomi setup.
If another LAN service needs the port, restrict it to the LAN and create no
router port-forward.

## 5. Sign in to Xiaomi and load devices

1. Open **Add → Xiaomi** in the go2rtc Web UI.
2. Enter the Xiaomi username and password.
3. Enter the SMS/e-mail code and solve CAPTCHA if requested.
4. Select the correct account region.
5. Click **load devices**.
6. Save the generated entries to the go2rtc configuration.

The upstream format looks like this:

```yaml
xiaomi:
  "<XIAOMI_ACCOUNT_ID>": "<ENCRYPTED_OR_SESSION_VALUE>"

streams:
  living_room:
    - "xiaomi://<ACCOUNT_ID>:<REGION>@192.0.2.10?did=<DEVICE_ID>&model=<MODEL>"
```

This is a schema, not a value to copy. Use the exact URL generated for your
camera. Do not guess the region, device ID, or model.

## 6. Quality, lenses, and `subtype`

The Xiaomi `miss` protocol uses quality values `0–5`. Typically:

- `0` or `auto` chooses automatically;
- `1` or `sd` selects lower quality;
- `2` or `hd` selects HD;
- some newer cameras place HD on another value;
- some older cameras return broken codec parameters on a forced high profile.

Examples:

```yaml
streams:
  camera_auto:
    - "xiaomi://<REDACTED>&subtype=auto"
  camera_sd:
    - "xiaomi://<REDACTED>&subtype=sd"
  dual_camera_second_lens:
    - "xiaomi://<REDACTED>&channel=2"
```

In the validated deployment, four cameras from one family became reliable after
restoring the exact model parameter and adding `subtype=auto`. Do not apply that
setting to every model without testing.

## 7. Validate one camera first

Before adding a multi-camera dashboard, prove that one stream:

1. appears on the go2rtc home page;
2. starts through WebRTC or MSE;
3. exposes the expected video and audio codecs to `ffprobe`;
4. has the expected resolution;
5. advances for 30–60 seconds rather than showing one frame;
6. releases the camera session when the viewer closes;
7. returns after restarting only the go2rtc add-on.

Example local probe without recording:

```bash
ffprobe -v error -show_streams \
  'rtsp://127.0.0.1:8554/living_room'
```

Use the Home Assistant UI for remote viewing instead of exposing RTSP.

## 8. Avoid opening every stream at once

Every active player creates a camera session, decoder, and network load. Xiaomi
P2P may also limit concurrent sessions. The first failed dashboard created
twelve players simultaneously, which increased latency and made dark players
hard to diagnose.

A reliable layout uses:

- one indoor-camera card;
- one active stream inside that card;
- a selector plus previous/next buttons;
- one default camera;
- a separate outdoor card when needed;
- independent state for the two cards;
- player removal when the popup or camera view closes.

## 9. Ingress and `401 Unauthorized`

A `/api/hassio_ingress/...` URL contains temporary context. Persisting that URL
in a card often produces `401 Unauthorized` after the app or browser restarts.

The working custom element created a native Ingress session through the Home
Assistant WebSocket API, set the cookie, then requested the add-on's current
`ingress_url`:

```javascript
const session = await hass.callWS({
  type: "supervisor/api",
  endpoint: "/ingress/session",
  method: "post",
});

document.cookie = [
  `ingress_session=${session.session}`,
  "path=/api/hassio_ingress/",
  "SameSite=Strict",
  location.protocol === "https:" ? "Secure" : "",
].filter(Boolean).join(";");

const app = await hass.callWS({
  type: "supervisor/api",
  endpoint: "/addons/a889bffc_go2rtc/info",
  method: "get",
});

const base = app.ingress_url.replace(/\/$/, "");
```

`a889bffc_go2rtc` is the public add-on repository slug, not a private home ID.
Still, discover the actual slug in your installation.

This pattern is specific to HAOS/Supervised and relies on Supervisor behavior;
retest it after Home Assistant updates. Prefer a maintained card's supported
Home Assistant/go2rtc integration when it provides the same behavior.

## 10. Load VideoRTC and autoplay

After obtaining the current `base`:

```javascript
const { VideoRTC } = await import(`${base}/video-rtc.js`);

class XiaomiLivePlayer extends VideoRTC {
  oninit() {
    super.oninit();
    this.video.muted = true;
    this.video.autoplay = true;
    this.video.playsInline = true;
    this.video.style.objectFit = "contain";
  }
}
```

iOS and Chromium autoplay is most reliable with `muted=true`. Enable sound only
after a user gesture.

When changing cameras, invalidate the previous async request, remove the old
player, create one new player, and ignore late responses from the old request.
On close, clear timers and remove the player instead of merely hiding it:

```javascript
stop() {
  clearInterval(this.sessionTimer);
  clearTimeout(this.checkTimer);
  this.player?.remove();
  this.player = null;
}
```

## 11. Validate long Ingress sessions

The validated component periodically checked the session:

```javascript
this.sessionTimer = setInterval(() => {
  hass.callWS({
    type: "supervisor/api",
    endpoint: "/ingress/validate_session",
    method: "post",
    data: { session: session.session },
  }).catch(() => this.showReconnectMessage());
}, 60_000);
```

Always clear the previous timer before creating another one.

## 12. Zoom around the touched point

Plain `transform: scale()` zooms around the center. Preserve the selected point
by recalculating translation:

```javascript
function zoomAt(nextScale, focusX, focusY) {
  const next = Math.max(1, Math.min(8, nextScale));
  const ratio = next / scale;
  panX = focusX - (focusX - panX) * ratio;
  panY = focusY - (focusY - panY) * ratio;
  scale = next;
  paint();
}
```

Track active `PointerEvent` objects for two-finger gestures. Pinch distance
changes scale, the two-point center is the focus, center movement pans, and a
single pointer pans only when scale exceeds one. Apply `touch-action: none` only
inside the video viewport and clamp translation.

Fullscreen should move the same player DOM node rather than create a second
stream. Return it to the card when fullscreen closes.

## 13. Codecs and browser compatibility

Some Xiaomi cameras expose H.265, PCMA, or another combination that a browser
cannot play directly. One validated camera decoded H.265 to a full-resolution
image and produced PCMA packets, although browser audibility was not separately
accepted.

If go2rtc sees the stream but the browser remains dark:

1. inspect the actual codecs in `api/streams`;
2. probe the stream with `ffprobe`;
3. try another VideoRTC/MSE transport;
4. add an **on-demand** H.264/AAC compatibility source only for the affected
   camera;
5. measure CPU before expanding transcoding.

Do not transcode every camera continuously when nobody is watching.

## 14. Observed failures and fixes

### Web UI remains on `Loading…`

Probe `/`, `/main.js`, `/add.html`, `/api`, and `/api/streams` separately. Record
status, `Content-Type`, and whether a real body is returned. Use browser
Network/Console to find the first broken resource. Compare local HA and external
HTTPS before reinstalling anything.

### A fresh browser gets `401`

The card stored an old Ingress URL or lacked a current session. Create and
validate a session before loading VideoRTC, obtain the current `ingress_url`, and
never persist the tokenized path in dashboard YAML.

### The player is dark or spins forever

Validate the stream in go2rtc and `ffprobe`, then check browser codec support.
Confirm that manual copying did not lose `model`, `did`, region, or `subtype`.

### `no-p2pid`

This occurred on two cameras that were physically disconnected. Other possible
causes include unsupported firmware/model, missing Xiaomi P2P credentials,
wrong region/account/device parameters, or a session limit. Avoid an infinite
retry loop. Remove the camera from the visible selector and retest it separately.

### A camera works only with `subtype=auto`

Some models returned invalid codec parameters on a forced high profile. Restore
the URL produced by go2rtc and test `subtype=auto` only for that model family.

### Twelve cameras fail when opened together

Too many P2P sessions, decoders, and DOM players were created. Switch to one
selectable stream per card. Keep separate state and timers for two independent
cards.

### Playback stutters after an optimization

A reduced-stream/recording pilot made the live view worse. Restore the original
source, natural playback rate, and full quality, and stop the recording pilot.
Change one variable at a time and compare resolution, advancing time, frame
count, and CPU.

### Sound does not autoplay

Browsers block unmuted autoplay. Start muted, enable audio after a user gesture,
and separately confirm that the audio codec is supported.

### Direct playback works but external HA does not

The problem is then in Ingress session handling, cookies, HTTPS, reverse proxy,
or asset URLs. Compare HTML, JavaScript, API, and WebSocket/RTC requests on local
and external routes.

## 15. SD-card archive and server recording

A live Xiaomi source in go2rtc does not imply access to recordings on the
camera's SD card. The validated setup did not import the Xiaomi Home archive.

Before server recording, measure one stream for at least 15–60 minutes during
representative day and night conditions. One short pilot estimated about **39.7
GB per 72 hours** for one particular stream. This is not a universal rate.

```text
GB per day = average bitrate in Mbit/s × 10.8
```

Add 20–30% reserve and measure CPU/RAM. Use Frigate or another NVR as a separate
project for recording and detection. Do not combine live-view integration,
recording, and AI detection in one change.

## 16. Acceptance checklist

- [ ] The default camera autoplays without a Watch/Play button.
- [ ] Time and frame count advance.
- [ ] Resolution matches the selected quality.
- [ ] Switching removes the old player.
- [ ] Two cards, if used, play independently.
- [ ] Pinch-to-zoom follows the point between the fingers.
- [ ] Panning works after zooming.
- [ ] Fullscreen preserves playback and gestures.
- [ ] Closing the camera view leaves zero video players.
- [ ] Reopening starts the default camera again.
- [ ] An expired Ingress session shows a useful reconnect state.
- [ ] An offline camera does not trigger an infinite retry loop.
- [ ] Local and external HTTPS routes behave consistently.
- [ ] Other Home Assistant dashboards and Frigate remain unchanged.

## 17. Rollback

Before editing, save the go2rtc config, dashboard JSON/YAML, card JavaScript,
stream list, file hashes, and a text description of expected behavior.

To roll back:

1. restore the previous JS and its resource version URL;
2. restore the previous dashboard card;
3. restore only the changed `streams` entries;
4. restart go2rtc only if its config changed;
5. avoid a full Home Assistant restart unless required;
6. repeat the one-camera probe and dashboard regression checks.

## 18. Publication hygiene

Never publish Xiaomi credentials or verification codes, the working `xiaomi:`
block, complete `xiaomi://` URLs, DID/MAC/IP/serial data, personal or room names,
camera frames, Ingress URLs, Home Assistant tokens, personal hostnames, or a
private dashboard dump.

Use placeholders such as `living_room`, `camera_a`, `192.0.2.10`,
`<DEVICE_ID>`, and `<REDACTED>`.

## Sources

- [go2rtc: Xiaomi Mi Home source](https://github.com/AlexxIT/go2rtc/blob/master/internal/xiaomi/README.md)
- [go2rtc repository](https://github.com/AlexxIT/go2rtc)
- [Home Assistant go2rtc add-on manifest](https://github.com/AlexxIT/hassio-addons/blob/master/go2rtc/config.yaml)
- [Home Assistant add-on Ingress](https://developers.home-assistant.io/docs/add-ons/presentation/)
- [Home Assistant WebSocket API](https://developers.home-assistant.io/docs/api/websocket/)

## License

Documentation is released under the [MIT License](LICENSE).
