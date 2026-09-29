# Identity Presence API

A small, self-hosted **Discord presence API** — live status, activities, and now-playing data for any Discord user — open for anyone to use. It is the service behind the live blocks (Identity panel, Spotify now-playing, Discord activity) on [identities.dev](https://github.com/Silver-Dev-Studios/identities.dev).

It speaks the same wire protocol as [Lanyard](https://github.com/Phineas/lanyard) (`GET /v1/users/:id` + `wss://…/socket`), so any client that already speaks Lanyard works against it unchanged.

**No API key. No token. No sign-up.** The public instance is CORS-open (`Access-Control-Allow-Origin: *`), so browser pages on any origin can call it directly.

## Endpoints

All routes hang off `https://identities-presence-uwui.onrender.com` (`wss://` for the socket). Every response is JSON.

| Method | Endpoint | What it does |
|---|---|---|
| `GET` | `/` | One-line reminder of the service endpoints. |
| `GET` | `/health` | Service status: gateway connection state + how many users are monitored. |
| `GET` | `/v1/users/:id` | A snapshot of a user's Discord presence. |
| `GET` | `/v1/guilds/:gid/members/:uid` | Check whether a user belongs to a guild (the gate behind the site's presence blocks). |
| `WS` | `/socket` | Live presence push channel — real-time updates. |

### Who can be looked up

A bot watches presence, and it can only see users who **share a guild with it**. On the public instance that means members of the [identities.dev community Discord server](https://discord.gg/vubY4SerXQ). If a user isn't a member, lookups return:

```json
{ "success": false, "error": { "code": "user_not_monitored", "message": "User is not being monitored" } }
```

That's the only gate — no key, no token, no rate limit. Join the server and the bot starts watching you within about a minute.

## Quick start

```bash
curl https://identities-presence-uwui.onrender.com/v1/users/1360925264669966338
```

```json
{
  "success": true,
  "data": {
    "discord_user": {
      "id": "1360925264669966338",
      "username": "alwaystrey",
      "discriminator": "0",
      "global_name": "Spooky Clouded",
      "avatar": "bb3098f1a8e7b92ccd892a1c60869b96",
      "public_flags": 0
    },
    "discord_status": "online",
    "activities": [],
    "listening_to_spotify": false,
    "spotify": null,
    "active_on_discord_web": true,
    "active_on_discord_desktop": false,
    "active_on_discord_mobile": false,
    "kv": {},
    "recv_at": 1790714766623
  }
}
```

Replace the user id with the snowflake (numeric ID) of anyone you want to watch. Neither the API nor the site exposes Discord tokens — you only need the user's public numeric ID (right-click a user → *Copy User ID*, or `@me` for yourself).

## Reading presence (REST)

### `GET /v1/users/:id`

A successful response wraps the data in a `data` object:

| Field | Meaning |
|---|---|
| `discord_user` | `{ id, username, discriminator, global_name, avatar, public_flags }` |
| `discord_status` | `"online"` \| `"idle"` \| `"dnd"` \| `"offline"` |
| `activities` | Raw Discord activity objects (games, rich presence, custom status). |
| `listening_to_spotify` | `true` while Spotify is playing. |
| `spotify` | `{ track_id, timestamps, song, artist, album_art_url, album }` or `null`. Album art resolves to an `i.scdn.co` URL. |
| `active_on_discord_web` | `true` when the user is online on web. |
| `active_on_discord_desktop` | `true` when online on desktop. |
| `active_on_discord_mobile` | `true` when online on mobile. |
| `kv` | Reserved key/value bag (always `{}` for now). |
| `recv_at` | Unix ms timestamp of the last gateway update. |

**Status codes**

| Code | Meaning |
|---|---|
| `200` | `{ success: true, data }` |
| `400` | `{ success: false, error: { code: "invalid_id", … } }` — an id that isn't a numeric snowflake. |
| `404` | `{ success: false, error: { code: "user_not_monitored", … } }` — the service can't see that user. |

### `GET /v1/guilds/:gid/members/:uid`

Returns whether a user is a member of a guild — useful to gate a widget or badge on server membership:

```bash
curl https://identities-presence-uwui.onrender.com/v1/guilds/1537587540281000067/members/1360925264669966338
```

```json
{ "success": true, "data": { "member": true, "guildId": "1537587540281000067", "userId": "1360925264669966338" } }
```

## Live updates (WebSocket)

For anything that should update in real time — a status dot, a now-playing card — hold one connection open and subscribe. The service speaks the Lanyard socket protocol:

| Direction | Opcode | Purpose |
|---|---|---|
| recv | `{ "op": 1 }` | **Hello** — `d.heartbeat_interval` (30s). Start heart-beating now. |
| send | `{ "op": 2,"d": { "subscribe_to_id": "…" } }` | **Initialize** — subscribe to one id, an `subscribe_to_ids` array, or `{ "subscribe_to_all": true }`. |
| send | `{ "op": 3 }` | **Heartbeat** — once per `heartbeat_interval`. |
| recv | `{ "op": 0,"t": "INIT_STATE", "d": { <id>: presence } }` | Snapshot of everything you subscribed to (map of id → presence). |
| recv | `{ "op": 0,"t": "PRESENCE_UPDATE", "d": presence }` | A user changed — `d` is one presence object. | 

```js
const ws = new WebSocket("wss://identities-presence-uwui.onrender.com/socket");

ws.onmessage = (e) => {
  const m = JSON.parse(e.data);
  if (m.op === 1) {            // Hello: subscribe + start heartbeat
    ws.send(JSON.stringify({ op: 2, d: { subscribe_to_id: USER_ID } }));
    setInterval(() => ws.send(JSON.stringify({ op: 3 })), m.d.heartbeat_interval);
  } else if (m.op === 0) {     // INIT_STATE or PRESENCE_UPDATE
    const presences = m.d;      // for INIT_STATE: { id: presence, … }; for UPDATE: one presence
    Object.values(presences).forEach((p) => render(p));
  }
};
```

Expect the connection to drop occasionally — reopen and re-subscribe on `close` (the site's own blocks reconnect with exponential backoff). A REST primer + WS is also fine: fetch once to paint fast, then let updates flow in.

## Try it yourself

**A live status pill** — paste this into any HTML page that wants a tiny live Discord-presence badge:

```html
<span id="status">…</span>
<script>
  const pill = document.getElementById("status");
  function paint(p) {
    const dot = { online: "🟢", idle: "🟡", dnd: "🔴", offline: "⚫" }[p.discord_status] || "⚫";
    pill.textContent = dot + " " + p.discord_user.username;
  }
  fetch("https://identities-presence-uwui.onrender.com/v1/users/" + USER_ID)
    .then((r) => r.json())
    .then(({ success, data }) => success && paint(data));
</script>
```

Because the API is CORS-open, this works from any origin — no proxy needed.

**Spotify now playing** in a few lines:

```js
const { data } = await (await fetch(
  "https://identities-presence-uwui.onrender.com/v1/users/" + USER_ID
)).json();

if (data && data.listening_to_spotify && data.spotify) {
  console.log("Now playing:", data.spotify.song, "by", data.spotify.artist);
  console.log("Album art:", data.spotify.album_art_url);
}
```

**From Python** (or any language — it's plain JSON over HTTPS):

```python
import urllib.request, json

with urllib.request.urlopen(
    "https://identities-presence-uwui.onrender.com/v1/users/" + USER_ID
) as r:
    body = json.load(r)

if body["success"]:
    print(body["data"]["discord_status"])
```

## Run your own instance

The service is a single Node file — `server/presence.js` in the [identities.dev repo](https://github.com/Silver-Dev-Studios/identities.dev). Running your own lets you watch users in **your** servers (the public instance only watches the identities.dev community server):

```bash
git clone https://github.com/Silver-Dev-Studios/identities.dev.git
cd identities.dev
npm install
DISCORD_BOT_TOKEN=<your-bot-token> node server/presence.js
```

| Variable | Required | Description |
|---|---|---|
| `DISCORD_BOT_TOKEN` | yes | A Discord **bot** token with the **Presence Intent** + **Server Members Intent** enabled, added to the servers whose users you want to track. |
| `PORT` | no | Service port (Render sets this; default `8089`). |
| `REQUIRE_GUILD_ID` | no | Guild users must join before they can add presence blocks (defaults to the identities.dev community server; empty = no gate). |
| `PUBLIC_BASE` | no | Canonical public URL of the service (defaults to the hosted instance). |
| `PRESENCE_API` | no | Discord REST base for tests (defaults `https://discord.com/api`). |

The bot only sees users it shares a guild with — that model is inherent to Discord presence, not a limitation of this code. Enable the intents in the [Discord developer portal](https://discord.com/developers/applications), add the bot to your server(s), and you're done.

## Good to know

- **Public + CORS-open.** `Access-Control-Allow-Origin: *` (methods `GET, OPTIONS`), so browser pages on any origin can call it directly.

- **Monitored = shares a guild with the bot.** Joining the [community Discord](https://discord.gg/vubY4SerXQ) is the only gate on the public instance — the bot starts watching you within about a minute. Users who aren't members return `user_not_monitored`. (A user whose identity is known but whose presence is unseen can come back as an `offline` snapshot instead.)

- **Free service can idle.** Cold starts take a few seconds — treat slow first fetches and WS reconnects as normal. If the service is down, patch with a short cache and retry.

- **The site's blocks use the same protocol.** The identities.dev live blocks (`lanyard`, `spotify`, `activity`) and the site's bot slash commands answer with this service's data — everything over the same op-based protocol.

- **No keys, no tracking.** Reads are anonymous and untracked — no auth to speak of. The service caches presence in memory per user and holds one subscription per connected client; it doesn't log or expose anything else.

## Also in the identities.dev ecosystem

identities.dev runs a second, smaller public API — the **identity store** — that serves the published profile pages (`/u/<username>`, `/c/<username>`) plus the likes, library, and updates endpoints. It is served by the site at `https://identities-dev.com/api/*` (same-origin with the SPA; CORS-open):

| Method | Endpoint | What it does |
|---|---|---|
| `GET` | `/api/health` | Service status + configured repos/branch. |
| `GET` | `/api/u` | List every published profile (`{ username, slug, path, sha, size }`). |
| `GET` | `/api/u/:username` | Read a user's primary published profile (public). |
| `GET` | `/api/u/:username/profiles` | List a user's profiles with titles + URLs (public). |
| `GET` | `/api/u/:username/:slug` | Read one of a user's extra (slugged) profiles (public). |
| `PUT` | `/api/u/:username` | Publish (create/update) your primary profile — needs your **Discord OAuth token** as `Bearer`. |
| `DELETE` | `/api/u/:username` | Unpublish your primary profile (Discord bearer). |
| `GET` | `/api/likes` | Like counts for every profile (public). |
| `GET` | `/api/u/:username/likes` | One user's like counts (public). |
| `GET` | `/api/u/:username/likes/:slug` | One profile's like count + whether you liked it (public; auth optional). |
| `POST` | `/api/u/:username/like` | Toggle your like on a profile (Discord bearer). |
| `GET` / `PUT` | `/api/library/:username` | Sync your draft library (token-gated — drafts aren't public). |
| `GET` | `/api/presence/:id` | Same-origin proxy of the presence snapshot above. |
| `GET` | `/api/presence/:id/member` | Same-origin proxy of the guild membership check. |
| `GET` | `/api/updates` | Public changelog/updates feed. |

Writes (publish, like, library) authenticate with the **Discord OAuth access token** from signing in at identities.dev — the service validates it against Discord and only writes under your own username (you can't touch anyone else's profiles). Reads are anonymous and public.

Everything in this doc was verified against the live public service and the source in `Silver-Dev-Studios/identities.dev` (`server/presence.js`).
