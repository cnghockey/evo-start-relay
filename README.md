# evo-start-relay

Control an **EvoStart / Fortin EVO-START LTE** remote starter from your smart-home setup — **Home Assistant, Homebridge, Node-RED, an AI agent, a voice assistant, or a plain `curl`**. It exposes your car as a small **token-gated HTTP API** (and, optionally, as native smart-home devices). Reverse-engineered from the EvoStart Android app, **zero npm dependencies** (Node `net` + `crypto` + `http` only).

> The EvoStart phone app is the only official way to remote-start these cars, and it has no public API, no smart-home integration, and no web UI. This is a self-hosted bridge that talks the app's own cloud protocol so *you* can automate your own vehicle.

```
Home Assistant / Homebridge / Node-RED / a voice assistant / curl / your agent
          ->  this relay (your server)  ->  EvoStart / ZXY268 cloud  ->  your car
   e.g.  curl "https://you.example.com/cmd/start?token=..."
```

## Compatible apps / devices
This talks to the **"hyll" / ZXY268** telematics platform, which is white-labeled under many brands. Apps known to run on the same `com.hyll` SDK + servers (`zxy268.com`, `htrack.cn`, `topm2m.net`, `vsmart.top`) include:

| App | Android package | Notes |
|---|---|---|
| **EvoStart** / **EvoStart 2** | `com.fortin.evostart1` / `com.fortin.evostart2` | Fortin. EvoStart: tested with this code. EvoStart 2: not tested yet |
| **iStartCar** | `com.hyll.istartcar` | remote start / car |
| **CYHY** | `com.hyll.cyhy2` | China-market rebadge (source of the `nvhiuzz11/cyhy` dump) |
| **SwapTrailer** | `com.hyll.swaptrailer` | trailer / asset tracking, same platform |

**Is yours compatible?** If your remote-start/GPS app decompiles to a `com.hyll.*` package, or its traffic goes to one of those domains, this code should work — you just need your account's login, server, terminal id, and (maybe) command codes (see below). The regional server and control codes vary per install, so capture your own.

## ⚠️ Safety & legal
- **A remote start turns on a real engine.** Never start a vehicle in an enclosed space (garage) — carbon-monoxide risk. Only automate a car parked somewhere safe to run.
- This talks to a third-party cloud (the "hyll"/ZXY268 telematics platform behind EvoStart) **using your own account credentials**, for interoperability with hardware you own. It may break when the vendor changes their app/API. Use at your own risk; you are responsible for how you use it.
- Not affiliated with Fortin or the platform operators.

## What it is
The EvoStart LTE module is a rebadge of a Chinese telematics platform ("hyll", servers under `zxy268.com` / `htrack.cn` / `topm2m.net`). The app speaks a **plaintext-TCP** protocol to a command server:

```
frame = 8 ASCII digits (byte length of body) + body
body  = {sys_head, app_head, body} JSON  +  HMAC-SHA1(txnkey, json) suffix
login  (code 100010): name + SHA-1(password); returns uid / ssk / txnkey
command (30xxxx):      signed with the session txnkey; body carries your terminal id
```

`src/zxy.js` is a small client for that protocol; `src/server.js` keeps one session warm and exposes it as HTTP; `src/smarthome.js` is an optional self-hosted Google Smart Home integration.

See **[LESSONS.md](LESSONS.md)** for the reverse-engineering write-up and the (many) Google-Home gotchas.

## Quick start
```bash
git clone https://github.com/cnghockey/evo-start-relay
cd evo-start-relay
cp .env.example .env && chmod 600 .env    # fill in the values (below)
npm start                                 # no deps to install
curl "http://127.0.0.1:3300/cmd/status?token=$RELAY_SECRET"
curl "http://127.0.0.1:3300/cmd/start?token=$RELAY_SECRET"   # starts the engine!
```
Endpoints: `/cmd/start|stop|lock|unlock|trunk|find|status`, plus `/health`. The relay binds `127.0.0.1` — put it behind a TLS reverse proxy (nginx/Caddy) if you expose it.

## Getting the auth (login, server, tid, codes)
All device-specific. You need Android **Platform Tools** (`adb`); **no root required**. Replace `<pkg>` with your app's package (`com.fortin.evostart1` for EvoStart, `com.hyll.istartcar` for iStartCar, etc.).

**0. Login — `EVOSTART_USER` / `EVOSTART_PASS`.** The *same email/phone + password you use to sign into the app.* That is your cloud auth — the relay logs in as you. Note: the platform allows **one active session**, so your phone app is signed out while the relay is linked; just sign in again when you want the app.

**1. Enable USB debugging.** Phone → Settings → About → tap **Build number** 7× → Developer options → **USB debugging** on. Plug into a computer with `adb`, accept the prompt.

**2. Server — `ZXY_HOST`.** With the app open and logged in, read its live TCP connection:
```bash
adb shell 'UID=$(cat /proc/$(pidof <pkg>)/status | awk "/^Uid/{print \$2}");
           cat /proc/net/tcp /proc/net/tcp6 | awk -v u=$UID "\$8==u{print \$3}"'
```
Decode each `HEX_IP:HEX_PORT` — the 8 hex chars are the IP with its bytes reversed, the 4 after `:` are the port (e.g. `2D7100CB:0BEA` → `203.0.113.45:3050`). The entry on port **3050** is the command server — that IP is your `ZXY_HOST`.

Accounts are **region-sharded**, so find yours rather than assuming. Known servers: the app's default is `s1.zxy268.com` → `119.23.228.82` (China); a **North-American** account was observed on `190.92.236.56` (Huawei Cloud). The full discovered list of servers, ports, transaction codes and error codes is in **[LESSONS.md](LESSONS.md)** → *Discovered platform reference*.

**3. Terminal id — `EVOSTART_TID`.** `adb logcat | grep -i tcpstmsg` while the app polls; the long numeric id is your vehicle's `tid`.

**4. Command codes (optional).** Defaults: `start 303120 / stop 303125 / lock 303110 / unlock 303115`. Control codes vary by module wiring, so capture yours: `adb logcat | grep -E 'lzhgsTxn|sendtcp'` while tapping each button in the app, then set `ZXY_CODE_*` in `.env`.

**Optional — read the protocol yourself.** Pull the APK (`adb shell pm path <pkg>`, then `adb pull <path>`) and decompile with [jadx](https://github.com/skylot/jadx); all the logic is in the `com.hyll` package.

## Connect it to your smart home
The relay is just the **token-gated HTTP API** above, so it drops into anything.

### Home Assistant (recommended — one integration, every ecosystem)
Point Home Assistant at the HTTP API and you inherit everything HA already bridges to (voice assistants, dashboards, HomeKit, automations). In `configuration.yaml`:
```yaml
rest_command:
  car_start:  { url: "https://you.example.com/cmd/start?token=YOUR_RELAY_SECRET" }
  car_stop:   { url: "https://you.example.com/cmd/stop?token=YOUR_RELAY_SECRET" }
  car_lock:   { url: "https://you.example.com/cmd/lock?token=YOUR_RELAY_SECRET" }
  car_unlock: { url: "https://you.example.com/cmd/unlock?token=YOUR_RELAY_SECRET" }
```
Wrap them in buttons/scripts (or a `switch`/`template` entity) and expose to whatever assistant you use.

### Homebridge / Node-RED / IFTTT / shell / AI agents
Anything that can make an HTTP request works — a `homebridge-http-switch`, a Node-RED `http request` node, an IFTTT webhook, a cron job, or an MCP tool an agent calls.

### Native cloud integration (optional, built-in)
The relay also ships a self-hosted **cloud-to-cloud** integration for assistants that want native devices without going through Home Assistant. It exposes **Car** (switch: on = start, off = stop) and **Car Lock** (switch: on = lock, off = unlock — a plain switch, not a `LOCK`, so automations can also *unlock*; see [LESSONS.md](LESSONS.md)).

> You invent `GH_CLIENT_ID` / `GH_CLIENT_SECRET` / `GH_LINK_SECRET` yourself (no third party to register with): set them in `.env` and enter the *same* values in your assistant's developer console. `GH_LINK_SECRET` is the password you type once, on the relay's own page, during linking.

Put the relay behind HTTPS on your own domain, then register a cloud-to-cloud integration with Authorization `…/oauth/authorize`, Token `…/oauth/token`, Fulfillment `…/smarthome`, and link it with your `GH_LINK_SECRET`. Then build routines with multiple trigger phrases. (Tip: many assistants block *unlock* as an automation action on a `LOCK` device — that's why "Car Lock" is a switch.)

## Security notes
- `.env` and `gh-tokens.json` hold secrets — keep them `chmod 600`, never commit (they're gitignored).
- The relay holds your EvoStart credentials and, while linked, holds the platform's single active session (your phone app will be logged out until you sign in again — the platform allows one session at a time).
- Gate `/cmd/*` with a long random `RELAY_SECRET`, and ideally IP-allowlist your reverse proxy.

## Credits
Reverse-engineered independently; the `com.hyll` package structure was the key. Related prior art for this platform: [`nvhiuzz11/cyhy`](https://github.com/nvhiuzz11/cyhy) (a Node ZXY268 experiment — note it omits the `%08d` length prefix and had login/hash mismatches; this project corrects those).

## More from us

- [Wandless VR Video Player](https://github.com/cnghockey/wandless-vr-video-player): watch VR180 and flat video on a Meta Quest with your hands alone. [Try it in the headset browser](https://cnghockey.github.io/wandless-vr-video-player/).
- [VLC for Android subtitle URL fix](https://github.com/cnghockey/vlc-android-subtitle-url-fix): VLC for Android shows the subtitles that Jellyfin for Android TV sends it.
- [Bestway Spa Relay](https://github.com/cnghockey/bestway-spa-relay): control a Bestway SmartSpa / Lay-Z-Spa from Home Assistant or Homebridge.
- [Sats4AI](https://sats4ai.com): AI tools that people and AI agents pay per call with Bitcoin over Lightning. No signup.

## Support / donate
This started as one stubborn *"why can't I just say **start my car**"* and escalated into reverse-engineering a telematics cloud. Worth it.

If you benefited from this and have a few sats to spare, you can send them to:

⚡ **`sats4ai@blink.sv`**

(It's a **Bitcoin Lightning** address — pay it from any Lightning wallet. Looks like an email, isn't one. Lightning only: no on-chain, no fiat, no refunds on gratitude.)

And consider visiting our site — more services for you and/or your clanker (yes, agents can pay for their own tools now): **[sats4ai.com](https://sats4ai.com)** — pay-per-call AI tools for Bitcoin, no signup.

_MIT licensed. Not affiliated with Fortin or the platform operators._
