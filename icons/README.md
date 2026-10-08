# Omat kuvakkeet

Tähän kansioon laitetaan pilotissa käytettävät kuvakkeet (esim. Luontoon.fi). Kansio liitetään generaattorikonttiin polkuun `/app/icons/custom`.

- Tiedostonimet: ks. `python-lipas2tak-tool/icons/README.md` (laavu.png, kota.png, …)
- Käyttöönotto: aseta `.env`:ssä `LIPAS2TAK_ICONS_DIR=/app/icons/custom` ja aja `docker compose up -d --force-recreate generator`
- Puuttuvat kuvakkeet korvataan automaattisesti oletussarjan kuvakkeilla
- PNG-tiedostoja **ei** viedä gittiin (`.gitignore`), koska Metsähallituksen symbolien käyttölupa on vielä auki
