# docker-lipas-integration

Docker Compose stack that runs the Lipas → TAK service:

| Service | Role |
|---|---|
| `generator` | [python-lipas2tak-tool](https://github.com/Street3r/python-lipas2tak-tool): fetches Lipas (and OpenStreetMap) data and produces data packages, KMZ files, KML network links and GPX routes weekly |
| `web` | nginx serves the files (`index.html` lists everything). TLS is handled by the server's reverse proxy |
| `publisher` | *(optional, profile `github`)* publishes a public version to a GitHub branch served by GitHub Pages |
| `feeder-*` | *(planned)* pushes selected areas into Data Sync missions. One container per TAK Server / group / certificate |

## Setup

```bash
git clone https://github.com/Street3r/docker-lipas-integration
cd docker-lipas-integration          # run the compose commands in this directory
cp .env.example .env
docker compose pull                  # prebuilt image ghcr.io/street3r/python-lipas2tak-tool:main
docker compose up -d generator
docker compose logs -f generator
```

The generator runs weekly at `LIPAS2TAK_SCHEDULE_AT` (default `mon 04:00`, time zone `LIPAS2TAK_TZ`, default `Europe/Helsinki`). On start it runs at once only if there is no earlier output or a scheduled run was missed, so restarts and updates do not move the schedule. To run now: `docker compose run --rm generator run --dump /data/raw` (the publisher picks up the new output).

To build the image yourself instead, clone python-lipas2tak-tool next to this repository, set `LIPAS2TAK_IMAGE=lipas2tak:local` in `.env` and run `docker compose build generator`.

Copy the results out of the volume: `docker compose cp generator:/data/output ./output`
(raw data: `generator:/data/raw`). Serve the files with nginx: `docker compose up -d web`.

The first run takes several minutes (Lipas paging, OSM, region boundaries). Raw data is saved in the volume (`/data/raw`) so problems can be investigated without fetching again.

Updating:

```bash
git pull --ff-only                   # this repository
docker compose pull && docker compose up -d
```

With a local build (`LIPAS2TAK_IMAGE=lipas2tak:local`): `git pull` in ../python-lipas2tak-tool, then `docker compose build generator && docker compose up -d --force-recreate generator publisher`.

### KML network links

In ATAK: **Import → KML Link**, paste the URL and enable auto refresh (e.g. 86400 s).

- All areas: `<PUBLIC_BASE_URL>/links/all_fi.kml` (or `links/all_en.kml`). National parks are on by default; regions are listed but off. Switch areas on and off in Overlay Manager → Network Resources.
- A single area: `<PUBLIC_BASE_URL>/fi/kansallispuistot/nuuksio/nuuksio_fi.kmz`.
- Icon set (install once): `<PUBLIC_BASE_URL>/iconset/lipas2tak_iconset.zip`.

`index.html` lists every URL.

**Internal network only:** set `PUBLIC_BASE_URL=http://<server LAN IP>:8080` and `WEB_BIND=0.0.0.0:8080` (or the LAN IP) in `.env`, then rebuild the outputs. The base URL is written into the link files, so changing it later means the links have to be re-added on the devices.

## Publishing to GitHub (optional)

The `publisher` container watches the generator output. After every run it builds a public version from the saved raw data and force-pushes it to the `data` branch of [finland-outdoor-tak-data](https://github.com/Street3r/finland-outdoor-tak-data). The public version always uses the default icon set, so custom icons are never published. The branch keeps only the latest version (a single commit). When no data changed, only the check time is updated, so the pages show when the data was last verified. Long pushes use ssh keepalive, and if a push is cut off the files are uploaded in batches.

1. Create a deploy key, readable by the container user (uid 10001):
   ```bash
   mkdir -p secrets
   ssh-keygen -t ed25519 -N "" -C lipas2tak-publisher -f secrets/github_deploy_key
   sudo chown 10001:10001 secrets/github_deploy_key && chmod 600 secrets/github_deploy_key
   cat secrets/github_deploy_key.pub
   ```
2. In the data repository, go to **Settings → Deploy keys → Add deploy key**. Paste the public key and tick **Allow write access**.
3. Recommended: protect `main` with **Settings → Rules → Rulesets → New branch ruleset** (target the default branch, *Restrict deletions* and *Block force pushes*). A deploy key's write access always covers the whole repository; this rule keeps `main` from being deleted or overwritten, and the publisher itself only ever pushes to `data`. (*Restrict updates* would also block the QR code workflow. On a free plan rulesets are enforced only once the repository is public.)
4. In `.env`, set `COMPOSE_PROFILES=github` and check the `PUBLISH_*` values.
5. Try it without pushing:
   ```bash
   docker compose pull
   docker compose run --rm publisher publish --raw /data/raw --workdir /data/publish --dry-run
   ```
6. Start it: `docker compose up -d publisher`, then `docker compose logs -f publisher`.

The publisher checks for new output every 10 minutes and publishes the first time it starts.

### GitHub Pages only (no nginx)

When the files are only served from GitHub Pages, the `web` service is not needed. Keep `WEB_BIND=127.0.0.1:8080` (or leave it as is) and start only:

```bash
docker compose up -d generator publisher
```

`PUBLIC_BASE_URL` then only affects the generator's local output, not the published version, which always uses `PUBLISH_BASE_URL`.

### Troubleshooting the push

- `Permission denied (publickey)`: the deploy key is missing from the data repository or lacks write access, or the key file is not readable by uid 10001.
- `Connection reset by peer` or `the remote end hung up unexpectedly` during a large push: some networks cut long SSH sessions on port 22. Use GitHub's SSH endpoint on port 443 in `.env`:
  `PUBLISH_REPO=ssh://git@ssh.github.com:443/Street3r/finland-outdoor-tak-data.git`
  then `docker compose up -d --force-recreate publisher`.

## Custom icons

By default the generator uses the tool's own icon set. To use another set, see [icons/README.md](icons/README.md).

## Feeders (planned)

Each feeder reads `index.json` and:
1. selects the areas and language from its configuration (e.g. `kansallispuistot/*`, `fi`)
2. creates or updates one Data Sync mission per area (TAK Server 5.8 API, client certificate)
3. adds new and changed CoTs and removes deleted ones (UIDs `lipas-<id>`, `loi-<uuid>`, `osm-<id>`)
4. restricts the mission to a group or channel

## Licence

- This repository: [MIT](LICENSE).
- The data served by this stack is licensed under [ODbL 1.0](https://opendatacommons.org/licenses/odbl/1-0/) and must be attributed: *Lipas – University of Jyväskylä (CC BY 4.0); © OpenStreetMap contributors (ODbL 1.0); Statistics Finland (CC BY 4.0)*. See [LICENSE-DATA.md](https://github.com/Street3r/python-lipas2tak-tool/blob/main/LICENSE-DATA.md) in python-lipas2tak-tool. The attribution is embedded in the generated files and shown in `index.html`; keep it when redistributing (e.g. via Data Sync).

---

## Suomeksi lyhyesti

Compose-pino ajaa python-lipas2tak-tool-generaattorin viikoittain (oletuksena maanantaisin klo 4.00 Suomen aikaa, valmiilla ghcr-kuvalla; päivitys `docker compose pull && docker compose up -d`) ja jakaa tuotokset nginxillä; Data Sync -feederit tulevat myöhemmin. Koodi on MIT-lisensoitu, ja jaettava aineisto on ODbL 1.0 -lisensoitu lähdemainintoineen (Lipas / Jyväskylän yliopisto, © OpenStreetMapin tekijät, Tilastokeskus).
