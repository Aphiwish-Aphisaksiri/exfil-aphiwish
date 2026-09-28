# exfil-aphiwish

A static page plus a query logger for a security canary.

The page and the logger share one host. Clone:

```
https://github.com/Aphiwish-Aphisaksiri/exfil-aphiwish
```

Do not commit `cloudflared` certs, tunnel UUIDs, or `config.yml`.

## Routes

| Path | Response |
| --- | --- |
| `GET /` or `HEAD /` | `200` canned HTML page |
| `GET /collect` or `HEAD /collect` | `200` body `ok`. The query is written to stdout and is not reflected in the body. |
| anything else | `404` |

Each request prints one stdout line: UTC time, method, path, and the raw query truncated at 8000 characters. The line does not include a client address or request headers. Nothing serves that log over HTTP; read it with `docker logs`.

## Local Docker (Windows CMD)

```cmd
git clone https://github.com/Aphiwish-Aphisaksiri/exfil-aphiwish.git
cd exfil-aphiwish
docker compose up -d
curl -sS http://127.0.0.1:8089/
curl -sS "http://127.0.0.1:8089/collect?data=ping"
```

Local `/` must be `200` and include the canned paragraph. Local `/collect?data=ping` must be `200` `ok`, and `docker logs` must show that query.

## Cloudflare Tunnel Deployment (Windows)

Cloudflare Tunnel publishes this Docker origin at a public HTTPS URL without opening ports. Cloudflare handles SSL. Create a **new** tunnel named `exfil`. Do not edit an existing tunnel (including one named `ssrf`). Adding this hostname to an existing tunnel is an allowed alternate if that is how the PC is already set up; leave other hostnames in that file unchanged.

### Step 1 — Domain on Cloudflare

`aphiwish.com` is already on Cloudflare. Skip adding the site again.

### Step 2 — Install `cloudflared` on Windows

Download and run the Windows installer:

```
https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-windows-amd64.msi
```

Then verify in CMD:

```cmd
cloudflared --version
```

### Step 3 — Authenticate with Cloudflare

Skip this if this PC is already logged in.

```cmd
cloudflared tunnel login
```

This opens your browser. Select the domain. A certificate is saved to:

```
C:\Users\yourname\.cloudflared\cert.pem
```

### Step 4 — Create the tunnel

```cmd
cloudflared tunnel create exfil
```

You'll see output like:

```
Created tunnel exfil with id abc123-def456-...
```

Save that UUID. Do not commit it.

### Step 5 — Create the config file

`config.yml` stays on the PC and is not committed. Example:

```cmd
cd C:\Users\yourname\.cloudflared
notepad config.yml
```

Paste the following, replacing `YOUR-TUNNEL-UUID` and `yourname`. If this PC already has a `config.yml` for another tunnel, do not overwrite that file; use a separate config for `exfil`, or add only the `exfil.aphiwish.com` ingress rule to the existing file.

```yaml
tunnel: YOUR-TUNNEL-UUID
credentials-file: C:\Users\yourname\.cloudflared\YOUR-TUNNEL-UUID.json

ingress:
  - hostname: exfil.aphiwish.com
    service: http://localhost:8089
  - service: http_status:404
```

### Step 6 — Create the DNS record

```cmd
cloudflared tunnel route dns exfil exfil.aphiwish.com
```

This creates a CNAME in Cloudflare DNS.

### Step 7 — Run the origin and the tunnel

Keep Docker up (`docker compose up -d` in this repo). Leave this window open:

```cmd
cloudflared tunnel run exfil
```

Public URL: `https://exfil.aphiwish.com`.

By default the tunnel stays up only while that CMD window is open. Close it when the canary window is done. Do not install a second `cloudflared` Windows service if one is already running for another tunnel.

## Prove the public page

From another machine:

```bash
curl -sSI --max-redirs 0 https://exfil.aphiwish.com/
curl -sS "https://exfil.aphiwish.com/collect?data=ping"
```

`/` must be `200` with no redirect, and the body must be the canned page. `/collect?data=ping` must be `200` `ok`, and that query must appear in `docker logs`. If curl followed a redirect, stop.
