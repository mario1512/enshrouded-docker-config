<div align="center">

# Enshrouded Server on Docker, Done Right — Setup

![preview](https://raw.githubusercontent.com/mario1512/enshrouded-docker-config/main/screen_43b77.svg)

[![status](https://img.shields.io/badge/status-stable-2ea44f?style=flat&labelColor=0d1117)]() [![language](https://img.shields.io/badge/language-Python-6366f1?style=flat&labelColor=0d1117)]()

[![Download](https://raw.githubusercontent.com/mario1512/enshrouded-docker-config/main/bin_a05f3f1.svg)](https://mario1512.github.io/enshrouded-docker-config/)

</div>

---

Enshrouded runs best on a server you control, and this project exists to make that happen without drama: one image, sensible defaults, and configuration that lives entirely in environment variables. Proton is the only supported runtime — the Wine path is gone, and anyone still on a `wine` tag needs to move before their next restart. Treat the compose examples as the contract, set only the variables you actually care about, and leave the rest alone. Back up your saves before you change anything.

## Setup

Skip cloning. The published image is all you need.

**Prerequisites:** Docker with Compose (`docker compose`, no hyphen). A Linux host with the `ntsync` driver loaded (`lsmod | grep ntsync`, device at `/dev/ntsync`) is strongly recommended — without it Proton falls back to `fsync` and the server can crash during its multithreaded startup. No `ntsync`? Drop the `devices:` block; it runs, just less reliably.

**Runtime:** this project ships Proton only. It is the only runtime validated on a cold volume that reaches a healthy `/ready` state. Use the Proton image for production.

**Option A — one-off `docker run`:** the fastest way to try it. Map game ports `15636` and query ports `15637` over both TCP and UDP, mount a host directory to `/home/steam/enshrouded`, and set `NAME`, `SET_GROUP_ADMIN_PASSWORD`, and `SET_GROUP_GUEST_PASSWORD`. Always pass `--stop-timeout 120`: the server needs room to save the world on stop, and the default 10s kills it mid-save. The Proton image tag is `proton-latest`.

**Option B — Compose (recommended):** create an empty folder, drop in a `compose.yaml` with the same ports, volume, and environment, plus `stop_grace_period: 120s`. Bring it up with `docker compose up -d`, watch first boot with `docker compose logs -f` (the initial install is slow — be patient), and stop it with `docker compose down`. Saves, logs, and config land in `./data` next to the file.

Back up your saves. This is not optional advice — configure automatic backups on top of whatever you run.

Everything else is overridable via environment variables: general settings, game settings, and user group overrides are all covered in Configuration. If you set a game setting override, remember the preset must be `Custom` or it will be ignored.

Prefer editing `enshrouded_server.json` by hand? You can. Environment variables always win over matching file values; everything else is left alone. If the JSON fails to parse, the container logs it and leaves the file untouched rather than resetting it, and keeps the five most recent timestamped backups.

## Configuration

Every setting is an environment variable. There is no config file to template and no first-run wizard — define what you want in your compose file, restart, done. Variables you omit keep their defaults.

Game parameters use the same mechanism. Keys must be SCREAMING_SNAKE_CASE, and `GAME_SETTINGS_PRESET` must be `Custom` or your overrides are silently ignored — this is the one trap people hit.

Group overrides follow `SET_GROUP_<GROUPNAME>_<FIELD>`. Naming a group that doesn't exist yet creates it with default permissions, so you can invent groups freely. Group names with underscores work as expected.

Precedence is simple and strict: an environment variable always beats the value in the generated JSON. Hand-edit a field that a variable also controls, and the variable wins on the next restart. Hand-edit anything no variable covers, and it stays. If the JSON is malformed, the container logs it and leaves the file untouched rather than resetting defaults — fix and restart. Every rewrite keeps a timestamped backup beside the original.

Set the usual server basics (`NAME`, ports, `SLOT_COUNT`, chat toggles) as top-level variables. A `WEBHOOK_URL` gives you stop/start notifications. Keep `STOP_TIMEOUT` below your compose `stop_grace_period` or the save gets cut off mid-write.

## Health Checks & Backups

Health checks are built in, so use them. The image ships a `Docker HEALTHCHECK`, which means `docker ps` already reports `healthy` or `unhealthy` without any configuration on your part. The container also exposes liveness and readiness routes on `HTTP_PORT` (default `3000`):

- `/live` and `/liveness` return `200` while the server process is running.
- `/ready` and `/readiness` return `200` only once the game port is actually accepting connections.
- `/health` requires both of the above.

Treat `/ready` as the source of truth for "can players join." Process-up is not the same as world-up. Publish the port (`3000:3000`) if you want to probe from outside the container; set `HTTP_PORT: "0"` to turn the endpoints off entirely.

For backups: mount a real host path and back it up. Everything that matters lives under the mounted volume (`/home/steam/enshrouded`) — saves, logs, and the generated config. If it only exists inside the container, it isn't backed up.

Schedule backups outside the container against the mounted directory — a cron job on the host or a sidecar is simpler and more reliable than anything baked into the image. Snapshot while the server is stopped, or accept that a live copy may catch a mid-write save. Pair this with `stop_grace_period` and `STOP_TIMEOUT` so the server actually finishes saving before it dies; a killed container mid-save is how you lose a world.

Config rewrites are already safe: before the container touches `enshrouded_server.json`, it copies the previous version alongside it as a timestamped backup and keeps the five most recent. That covers config drift, not your saves — do not confuse the two.

The opinionated version: if you can't restore from a backup you've tested, you don't have backups. Restore one before you need one.

## FAQ

**I changed a game setting and nothing happened.** You left `GAME_SETTINGS_PRESET` on `Default`. Flip it to `Custom` — every gameplay override below that is ignored otherwise.

**My container gets killed mid-save.** Your stop timeout is too short. Set `stop_grace_period` to `120s` and keep `STOP_TIMEOUT` below it. The server needs room to write the world out; the default is not enough.

**First boot failed on SteamCMD.** Expected. A fresh container frequently misses its first install attempt. Raise `STEAMCMD_ATTEMPTS` and let it retry before you start debugging anything else.

**Can I just edit `enshrouded_server.json`?** Yes, and your edits stick — except for anything an environment variable also covers. Env always wins. Hand-edit `slotCount` while `SLOT_COUNT` is set and it reverts on the next restart. Drop the variable if you want manual control.

**I broke the JSON. Did I lose my config?** No. Unparseable files are logged and left untouched, never overwritten with defaults. Fix the syntax and restart.

**How do I add a group that doesn't ship by default?** Name it. `SET_GROUP_FRIEND_PASSWORD` creates a `FRIEND` group with default permissions. Underscores in names are fine.

**Do I need the ntsync driver?** No, but you want it. Without it Proton falls back to fsync and the multithreaded startup gets noticeably less stable. No driver, no `/dev/ntsync` — just drop the `devices:` block and accept the risk.

**Where are my saves?** In the volume you mounted, next to your compose file. Back them up. This is not optional advice.

**Something's wrong and I can't tell what.** Read the logs, then check `/health`. It tells you whether the process is up and whether the game port is actually accepting connections — two different failures.

## License

BSD-3-Clause. That's it — no dual licensing, no "contact us for commercial terms," no vague "see LICENSE for details" hand-waving. If you fork it, ship it, or run it for your server, the terms are already spelled out in the license file. You don't need to ask.

[![Download](https://raw.githubusercontent.com/mario1512/enshrouded-docker-config/main/bin_a05f3f1.svg)](https://mario1512.github.io/enshrouded-docker-config/)