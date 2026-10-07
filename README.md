# docker-lipas-integration

Docker Compose -pino, joka pyörittää Lipas → TAK -palvelua:

| Palvelu | Tehtävä |
|---|---|
| `generator` | [python-lipas2tak-tool](https://github.com/Street3r/python-lipas2tak-tool): hakee Lipaksen datan ja tuottaa datapaketit, KMZ:t, KML-verkkolinkit ja GPX:t viikoittain |
| `web` | nginx jakaa tiedostot (`index.html` listaa kaikki). TLS hoidetaan palvelimen reverse proxyssa |
| `feeder-*` | *(tulossa)* vie valitut alueet Data Syncin missioihin. Yksi kontti per TAK Server / ryhmä / sertifikaatti |

## Käyttöönotto

```bash
cp .env.example .env          # aseta PUBLIC_BASE_URL
docker login ghcr.io          # jos image on yksityinen
docker compose up -d
docker compose logs -f generator
```

Ensimmäinen ajo kestää useita minuutteja (Lipas-sivutus, OSM, maakuntarajat). Raakadata tallentuu volyymiin (`/data/raw`), jotta ongelmat voi selvittää ajamatta hakua uudelleen.

KML-verkkolinkki ATAKiin: `https://<domain>/fi/kansallispuistot/nuuksio/nuuksio_fi.kml` (tai suoraan `.kmz`). Kuvakesarja: `https://<domain>/iconset/lipas2tak_iconset.zip`.

## Feederit (suunnitelma)

Jokainen feeder lukee `index.json`:n ja:
1. valitsee konfiguraationsa mukaiset alueet ja kielen (esim. `kansallispuistot/*`, `fi`)
2. luo tai päivittää Data Sync -mission per alue (TAK Server 5.8 API, client-sertifikaatti)
3. lisää uudet ja muuttuneet CoT:t ja poistaa poistuneet (UID = `lipas-<id>`, `loi-<uuid>`, `osm-<id>`)
4. rajaa mission ryhmään tai kanavaan
