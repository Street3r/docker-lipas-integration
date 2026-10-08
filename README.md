# docker-lipas-integration

Docker Compose stack that runs the Lipas → TAK service:

| Service | Role |
|---|---|
| `generator` | [python-lipas2tak-tool](https://github.com/Street3r/python-lipas2tak-tool): fetches Lipas (and OpenStreetMap) data and produces data packages, KMZ files, KML network links and GPX routes weekly |
| `web` | nginx serves the files (`index.html` lists everything). TLS is handled by the server's reverse proxy |
| `feeder-*` | *(planned)* pushes selected areas into Data Sync missions. One container per TAK Server / group / certificate |

## Setup

Clone both repositories side by side:

```bash
mkdir lipas && cd lipas
git clone https://github.com/Street3r/python-lipas2tak-tool
git clone https://github.com/Street3r/docker-lipas-integration
cd docker-lipas-integration          # run the compose commands in this directory
cp .env.example .env
docker compose build generator       # builds the image from ../python-lipas2tak-tool
docker compose up -d generator
docker compose logs -f generator
```

Copy the results out of the volume: `docker compose cp generator:/data/output ./output`
(raw data: `generator:/data/raw`). Serve the files with nginx: `docker compose up -d web`.

The first run takes several minutes (Lipas paging, OSM, region boundaries). Raw data is saved in the volume (`/data/raw`) so problems can be investigated without fetching again.

Updating after a `git pull` in either repository:

```bash
docker compose build generator && docker compose up -d --force-recreate generator
```

### KML network links

In ATAK: **Import → KML Link**, paste the URL and enable auto refresh (e.g. 86400 s).

- All areas: `<PUBLIC_BASE_URL>/links/all_fi.kml` (or `links/all_en.kml`). National parks are on by default; regions are listed but off. Switch areas on and off in Overlay Manager → Network Resources.
- A single area: `<PUBLIC_BASE_URL>/fi/kansallispuistot/nuuksio/nuuksio_fi.kmz`.
- Icon set (install once): `<PUBLIC_BASE_URL>/iconset/lipas2tak_iconset.zip`.

`index.html` lists every URL.

**Internal network only:** set `PUBLIC_BASE_URL=http://<server LAN IP>:8080` and `WEB_BIND=0.0.0.0:8080` (or the LAN IP) in `.env`, then rebuild the outputs. The base URL is written into the link files, so changing it later means the links have to be re-added on the devices.

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

Compose-pino ajaa python-lipas2tak-tool-generaattorin viikoittain ja jakaa tuotokset nginxillä; Data Sync -feederit tulevat myöhemmin. Koodi on MIT-lisensoitu, ja jaettava aineisto on ODbL 1.0 -lisensoitu lähdemainintoineen (Lipas / Jyväskylän yliopisto, © OpenStreetMapin tekijät, Tilastokeskus).
