# Hafez OpenWA

A self-hosted [OpenWA](https://github.com/rmyndharis/OpenWA) WhatsApp gateway for Hafez Gallery's apps:

- hafez_bot: the menu-driven WhatsApp auto-reply.
- [hafez_3d](https://github.com/mbadis-hafez/hafez_3d): the Mawsure Gallery Studio server (dashboard replies, booking messages, team alerts).

Run one gateway per WhatsApp number. Each app reaches it over the shared `openwa` Docker network.

OpenWA drives WhatsApp Web. It is not WhatsApp's official Business API. Use a number that belongs to the gallery and let a new number warm up (`SEND_PACING_ENABLED` is on).

## Setup

1. Start the gateway. The first run downloads an image of about 530 MB:
   ```bash
   docker compose up -d
   ```
2. Get the admin API key:
   ```bash
   docker compose exec openwa cat /app/data/.api-key
   ```
3. Open the dashboard at http://localhost:2785, create a session, and scan the QR code with the gallery's phone (WhatsApp › Settings › Linked devices).
4. Create an OPERATOR API key for each app in the dashboard, restricted to that app's session.

## Connecting an app

Add the app's container to the `openwa` network:

```yaml
services:
  my-app:
    networks:
      - openwa

networks:
  openwa:
    external: true
    name: openwa
```

Then, in that app:

- OpenWA's URL is `http://openwa:2785`.
- The webhook callback URL is the app's container name, for example `http://hafez-bot:3000/webhook`.

The container name must be listed in `SSRF_ALLOWED_HOSTS`, or OpenWA refuses to send webhooks to it. The default list is `hafez-bot,server,host.docker.internal`. To change it, set `SSRF_ALLOWED_HOSTS` in `.env` and run `docker compose up -d` again.

Start this gateway before the apps, because it creates the `openwa` network.

## Moving from the hafez_3d compose

The hafez_3d project used to run OpenWA itself. Its linked session is stored in the `3d_openwa` Docker volume. To keep that session instead of scanning the QR code again, copy the volume once while both gateways are stopped:

```bash
docker compose -p 3d stop openwa
docker volume create hafez-openwa_openwa
docker run --rm -v 3d_openwa:/from -v hafez-openwa_openwa:/to alpine cp -a /from/. /to/
docker compose up -d
```

Then remove the `openwa` service from hafez_3d's compose so the number isn't linked twice.

## Data

The `openwa` volume holds the WhatsApp session and message data. Back it up, and never commit it.
