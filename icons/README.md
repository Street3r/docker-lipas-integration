# Custom icons

Optional icon set for your own deployment. This folder is mounted into the generator container at `/app/icons/custom`.

- File names: see `python-lipas2tak-tool/icons/README.md` (laavu.png, kota.png, …)
- Enable: set `LIPAS2TAK_ICONS_DIR=/app/icons/custom` in `.env` and run `docker compose up -d --force-recreate generator`
- Missing icons fall back to the default set automatically
- The PNG files are **not** committed to git (`.gitignore`). Only use icons you have the right to use and distribute; they are embedded in the generated iconset and KMZ files.
