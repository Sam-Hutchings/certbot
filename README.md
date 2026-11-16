# Certbot via Docker Compose with Secrets

Docker image that issues a Let's Encrypt wildcard-free certificate for your chosen domain using the Cloudflare DNS-01 challenge. The Cloudflare API token is delivered to the container as a Docker Compose secret.

## How it works

- The image is built on the fly (`dockerfile_inline`) from `arm64v8/debian:13.7-slim`, installing `certbot` and `python3-certbot-dns-cloudflare` plus dependencies.
- `./cloudflare_credentials.ini` (host file, not committed) is declared as a Compose secret and mounted read-only inside the container at `/run/secrets/dns_cloudflare_api_token`.
- The container runs once (`restart: "no"`) and executes `certbot certonly --dns-cloudflare` against that secret file path.

## Project layout

```
.
├── compose.yaml    #Docker compose
└── cloudflare_credentials.ini      # Cloudflare API token (NOT COMMITED, added to .gitignore)
```

## Setup

**1. Create the credentials file** — `chmod 600`, exactly one line:

```ini
dns_cloudflare_api_token = <your-cloudflare-api-token-secret>
```

Paste the token **secret**, not the token ID.

**2. Configure the API token in Cloudflare** (*My Profile → API Tokens*). The token needs **both** permissions, scoped to the zone:

| Permission | Scope |
|---|---|
| Zone → DNS → Edit | domain.com |

Replace domain.com with your domain.

Cloudflare's built-in **"Edit zone DNS"** template pre-populates exactly these.

**3. Run:**

Replace domain.com with your domain in the compose.yaml file.
```bash
docker compose up --build
```

Certificates are written to `/etc/letsencrypt/live/domain.com/` **inside the container**. This requires a volume for `/etc/letsencrypt` to enable another service (nginx, adguard, reverse proxy) needs them, since the container is ephemeral.  
- Create the directory 'certs' with a sub-directory of 'live' and 'test'.
- Alternatively amend the volume mount below as required in the compose.yaml file.
```
    volumes:
      - type: bind
        source: ./certs/live
        target: /etc/letsencrypt/live
      - type: bind
        source: ./certs/archive
        target: /etc/letsencrypt/archive
```
## Verifying the token from the host

```bash
TOKEN=$(awk -F'= ' '/dns_cloudflare_api_token/{print $2}' ./cloudflare_credentials.ini)

# Is the token valid?
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.cloudflare.com/client/v4/user/tokens/verify"

# Can it see the zone? (tests Zone → Zone → Read)
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.cloudflare.com/client/v4/zones?name=domain.com"
```

## Renewal

Certbot skips renewal until ~30 days before expiry, so re-running the container periodically suffices:

```bash
docker compose up
```

For automation, schedule this with cron/systemd timer, or convert the service into a renewal loop.

## Troubleshooting notes
- **`Unable to determine zone_id`** — token lacks *Zone → Zone → Read*.
- **`10000 Authentication error` when adding TXT record** — token valid but lacks *Zone → DNS → Edit*.
- **`9109 Invalid access token`** — the string in the INI isn't a valid token secret (wrong token pasted, token rolled without saving the new secret, or file corruption). Verify with the curl above, using the token straight from the file.
- **Stale deployments** — if editing the compose file seems to have no effect, confirm with `docker compose config`

## Security notes
- Keep `cloudflare_credentials.ini` out of version control (`.gitignore` it) and `chmod 600`.
- Secrets are mounted as read-only tmpfs files; the token never appears in the compose file, process arguments, or `docker inspect` output.
- The Debian-packaged `cloudflare` Python library emits `PendingDeprecationWarning` noise about the 2.x→3.x transition — harmless for issuance.
