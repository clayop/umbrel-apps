# clayop Umbrel App Store

Community app store for umbrelOS. Add it in umbrelOS under
App Store → ⋯ → Community App Stores, using this repository's HTTPS URL.

## Apps

### Zebra (`clayop-zebra`)

Zcash full node ([zebrad](https://zebra.zfnd.org/)) on Mainnet, validating only (no mining).

| Service | Image | Role |
|---|---|---|
| `zebra` | `zfnd/zebra:6.4.2` (amd64 + arm64, pinned by digest) | Node. State in `${APP_DATA_DIR}/data/zebra` |
| `web` | `nginx:1.30-alpine` (pinned by digest) | Status page (via app_proxy) and authenticated RPC on the LAN |

Ports on the host:

| Port | Purpose |
|---|---|
| 8233/tcp | Zcash P2P. Conflicts with the official Linkwarden app (its UI uses 8233). |
| 8232/tcp | JSON-RPC, HTTP basic auth: user `zebra`, password = umbrelOS app password (`$APP_PASSWORD`). |
| 8234/tcp | Umbrel UI (status page). |

Design notes:

- zebrad cookie auth is disabled; zebrad's RPC port is not published. Only nginx on 8232
  reaches it, and it requires basic auth (`{PLAIN}` user file rendered from `$APP_PASSWORD`).
- The status page calls `/rpc` through the umbrelOS app_proxy, so the Umbrel login protects it.
  It polls every 5 s and judges health itself (tip age when synced, height progress while
  syncing): zebrad's `getinfo.errors` is sticky and keeps the last error after recovery, so
  that string is shown only while the node is unhealthy and only if raised after the problem began.
- The zebra image entrypoint runs as root, chowns the state dir to UID 10001 and drops
  privileges with `setpriv`, so the service has no `user:` override.
- Requirements: ~300 GB disk (growing), 16 GB RAM recommended, multi-day initial sync.

Example client call:

```sh
curl -u zebra:<password> -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getblock","params":["3000000",2]}' \
  http://umbrel.local:8232/
```

Updating Zebra: bump the tag and digest in `docker-compose.yml` and `version` in `umbrel-app.yml`
(`https://hub.docker.com/v2/repositories/zfnd/zebra/tags/<tag>` returns the index digest).
Packaging-only changes keep the Zebra version and add a revision suffix (`6.4.2-1`, `6.4.2-2`, ...);
umbrelOS offers an update whenever the store `version` string differs from the installed one.
