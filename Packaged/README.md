# Neon Strike — LAN Multiplayer

Play over your local network / Wi-Fi. No port forwarding, no public server, no accounts,
no matchmaking service. One PC runs a small Node server; everyone else opens a browser.

---

## Files

| File | What it is |
|---|---|
| `server.js` | Node server: serves the game files **and** runs the authoritative match |
| `package.json` | Dependency list (just `ws`) |
| `index.html` | The game |
| `net.js` | Client networking module (sockets, protocol, interpolation) |
| `three.min.js` | Three.js — **you must have this here already** |

All five files must sit in the same folder.

---

## Host: starting a game

**One time only**, open a terminal in this folder and run:

```
npm install
```

**Every time you want to play:**

```
npm start
```

(or `node server.js` — the same thing)

The server prints something like:

```
  ███ NEON STRIKE — LAN server running
  ----------------------------------------------------
  On this PC (the host):   http://localhost:3000
  Others on your network:  http://192.168.2.12:3000
  ----------------------------------------------------
```

Then:

1. Open **http://localhost:3000** in your browser.
2. Click **HOST**.
3. Pick a map with **CHANGE MAP** if you want (your choice applies to everyone).
4. The **JOIN ADDRESS** on screen is what other players type. Read it out to them.
5. Wait for players to appear in the list, then click **START GAME**.

You can press **Esc** at any time during the match — your join address and player
count stay visible in the pause menu, so late players can still be told how to connect.

**Different port?** `PORT=3001 npm start` (macOS/Linux) or `set PORT=3001 && npm start` (Windows).

---

## Other players: joining

Nothing to install. On any PC, phone or laptop on the same network:

1. Open the host's address in a browser — e.g. **http://192.168.2.12:3000**
2. Click **JOIN GAME**.
3. The address box is already filled in. Click **JOIN**.
4. Wait in the lobby. The match begins when the host clicks START GAME.

If someone opens the game from a file on their own machine instead, they can still join —
they just have to type the host's address themselves.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| "No answer from …" | The host's server isn't running, or you're on a different network (check for a guest Wi-Fi network, or one PC on Wi-Fi and the other on a VPN). |
| Host screen says to run the server | You opened `index.html` from disk. Open the `http://…` address the server printed instead. |
| "Port 3000 is already in use" | Another copy of the server is running, or something else uses that port. Use a different `PORT`. |
| Joiners can't connect at all | Your OS firewall is likely blocking Node. Allow it on **private networks**. |
| "This server already has a host" | Someone already clicked HOST on this server. Only one host per server. |

---

## Game modes

- **PLAY** — the normal single-player game, with the practice dummies. Unchanged.
- **HOST / JOIN GAME** — multiplayer. **No dummies at all**; combat is between real players.

The main menu still shows the live aerial camera over whichever map is selected, and the
host's map choice becomes everyone's map when the match starts.

---

## How the networking is arranged

`net.js` is the whole client networking layer and contains no gameplay code. `index.html`
has one clearly-marked `MULTIPLAYER INTEGRATION` section that connects it to the game;
nothing else in the game file knows the network exists.

**The server owns** health, damage, elimination, respawn timing, match phase, the selected
map and the player list. **Clients simulate their own movement** (they hold the collision
geometry) and the server validates and re-broadcasts it — a teleport gets rejected and the
client is snapped back.

**Shooting** reuses the existing raycast. Your client casts against its own world, then
reports *what it thinks it hit*. The server re-checks that claim against its own copy of
everyone's positions — range, whether the target is actually on the ray you fired, whether
you're firing faster than the weapon allows — and then computes the damage **from its own
weapon table**. A client that claims "I hit Player 2 for 500" gets that number thrown away.

Known limit, stated plainly: the server does not hold map geometry, so wall occlusion is
still trusted from the client. That's a deliberate trade for a LAN game — it avoids
duplicating every map on the server. The validation hooks are all in `handleShot()` in
`server.js`, which is where stricter checks (including server-side geometry) would go.

**Bandwidth**: snapshots go out 20×/second as compact arrays — about **2–3 KB/s per player**.
No scene data, geometry, textures or assets ever cross the network; every client already
has them. Remote players are interpolated ~110 ms in the past so they glide rather than
snap between packets.

### Adding to it later

- Internet play / room codes: `NeonNet.host()` and `NeonNet.join()` already take an
  arbitrary address; a relay or signalling step slots in there.
- Teams: `publicPlayer()` and the damage path in `server.js` already branch on a `team`
  field (`CONFIG.friendlyFire`).
- Dedicated server: nothing in `server.js` requires a host *player* — drop the
  `role: 'host'` requirement and it runs headless.
