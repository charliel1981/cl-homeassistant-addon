# Bitwarden Bridge (Home Assistant add-on)

Puts Charlie's own Bitwarden vault behind a small, token-guarded HTTP API inside Home Assistant, so the
`bitwarden_bridge` integration and the cl-homeassistant MCP tools can search, file and create logins. The bridge
contract it serves is in `.planning/plans/2026-11-22-bitwarden.md` ("Bridge contract").

Inside the container:

- the Bitwarden CLI, pinned at `@bitwarden/cli@2026.9.1`;
- on start, `bw config server`, `bw login --apikey` and `bw unlock --passwordenv BW_PASSWORD --raw`, then
  `bw serve --hostname 127.0.0.1 --port 8088` with the session in `BW_SESSION`. Secrets reach bw only through each
  process's environment, never its command line;
- a dependency-free Node front (`app/bridge.mjs`, plain `node:http`) on `0.0.0.0:8087`, the only way in. It re-unlocks
  when bw serve says the vault is locked, checks every minute, syncs every `sync_minutes`, and restarts bw serve if it
  exits. It logs method, path, status and time only: never a body, a password or the token.

Port 8087 is open on Home Assistant's internal network only (`ports: 8087/tcp: null`): no host port, no ingress.
Clients in HA reach it at `http://local-bitwarden-bridge:8087` when installed from the local folder, or
`http://<repo-hash>-bitwarden-bridge:8087` from the store (the add-on's hostname is on its Info page).

There are no delete routes, and trash is left out everywhere.

## Options

| Option | What |
|---|---|
| `server_url` | The vault: `https://vault.bitwarden.com` (default), `https://vault.bitwarden.eu`, or a self-host. Must be https: bw refuses anything else. |
| `client_id`, `client_secret` | The personal API key (below). |
| `master_password` | Unlocks the vault after every restart. |
| `api_token` | What clients send as `Authorization: Bearer <api_token>`. At least 32 characters, or the add-on will not start. Make it long and random (`openssl rand -hex 32`), its own value. |
| `sync_minutes` | How often `bw sync` runs, 5 to 1440 (default 15). |
| `folder_rules` | For `POST /house/tidy`: a list of `{match, folder}`. `match` is a host suffix (`octopus.energy` matches `my.octopus.energy` but not `notoctopus.energy`) or `name:<text>` (case-insensitive part of the item name). The first matching rule wins; only unfiled items are moved, and only when `apply` is true. |

### Where the API key comes from

Bitwarden web vault → **Account settings → Security → Keys → View API key**. It asks for the master password, then
shows `client_id` (starts `user.`) and `client_secret`. "Rotate API key" there invalidates the old pair.

## The risk

This add-on holds the **master password** in its options and in memory for as long as it runs, so it can unlock the
vault by itself after a restart. Anyone who can read the add-on's options (an HA admin, a full HA backup, the
Supervisor) or the container's memory can open the whole vault. Charlie chose this on 2026-11-22 so the vault comes
back on its own. The vault's data on disk (`/data/bitwarden`) stays encrypted as bw keeps it.

Keep HA backups encrypted, keep `api_token` out of anywhere it could be logged, and remember that anything revealed
through `reveal=1` (password, current TOTP code, notes) lands wherever the caller puts it, such as a Claude transcript.

To undo: uninstall the add-on (this removes `/data` and the bw login), then rotate the API key and, if in doubt,
the master password.

## Tests

From the repo root, no Bitwarden needed (a fake bw serve stands in):

```sh
node --test bitwarden-bridge/test/
```

## Build and push the image from the Mac

GitHub Actions are off, so the image is built here, like cl-homeassistant's (Docker Desktop running; gh has
`write:packages`). The folder is already a complete add-on build context, so there is nothing to stage.

```sh
gh auth token | docker login ghcr.io -u charliel1981 --password-stdin
V=0.1.0   # the version in bitwarden-bridge/config.yaml
cd bitwarden-bridge
docker buildx build --platform linux/amd64 --provenance=false \
  --build-arg BUILD_VERSION=$V --build-arg BUILD_ARCH=amd64 \
  -t ghcr.io/charliel1981/bitwarden-bridge-amd64:$V --push .
docker buildx build --platform linux/arm64 --provenance=false \
  --build-arg BUILD_VERSION=$V --build-arg BUILD_ARCH=aarch64 \
  -t ghcr.io/charliel1981/bitwarden-bridge-aarch64:$V --push .
```

Then in the store repo (`charliel1981/cl-homeassistant-addon`), add `bitwarden_bridge/config.yaml`: this folder's
`config.yaml` with an `image:` line after `version:`:

```sh
sed "s|^version: .*|&\nimage: ghcr.io/charliel1981/bitwarden-bridge-{arch}|" bitwarden-bridge/config.yaml \
  > <store>/bitwarden_bridge/config.yaml
```

Installing it in HA, and filling in the options, is Charlie's call.

To check a local build without pushing: `docker buildx build --platform linux/amd64 --load -t bitwarden-bridge:local .`
then `docker run --rm --entrypoint bw bitwarden-bridge:local --version`.
