# Setting Up Bluesky PDS

```
Updated at: 2025-09-09
Originally created at: 2024-11-25
```

I set up a Bluesky Personal Data Server (PDS) to host my own Bluesky data. I run it on my server with other apps and databases. I also wanted my Bluesky handle to be my root domain.

I started with the official PDS install guide. It worked at first, but the default config didn’t fit my setup and some containers failed to start.

I fixed the issues and got everything running smoothly. This guide shows what I changed so you can avoid the same problems.

### Domain Choices

For my setup, the PDS runs on a subdomain and my handle uses the root domain:

- PDS domain: `pds.skiddle.id`
- Handle domain: `skiddle.id`

This keeps the service separate and the handle clean. The examples below use this layout.

### Quick Start

1. Point `pds.skiddle.id` to your server (A/AAAA or CNAME). Enable TLS on your proxy.
2. Run the PDS with Docker Compose (see `compose.yml` below). Expose it on an internal port like `6010`.
3. From your public domain, proxy only these paths to the PDS:
   - `/xrpc` (include websocket headers)
   - `/.well-known/atproto-did` (forward the `Host` header)
4. If needed, set `PDS_SERVICE_HANDLE_DOMAINS` (example below).
5. Test the websocket connection:
   - `wss://pds.skiddle.id/xrpc/com.atproto.sync.subscribeRepos?cursor=0`
6. Create your account with `pdsadmin`. Then verify your handle on the root domain `skiddle.id`.
7. Optional: set up SMTP so email verification works.

## Manual Container Management

The default `installer.sh` mostly worked, but some containers didn’t start. When I tried to create an account with the script, I got 400/500 errors.

The default installer (as of November 2024) creates these Docker services via docker-compose:

- The PDS itself
- A Caddy reverse proxy
- containrrr/watchtower which seems to facilitate zero-downtime upgrades

It has code to auto-install Docker and some other packages, pull images, and launch containers. The compose.yml file that it uses can be found here: https://github.com/bluesky-social/pds/blob/main/compose.yaml

I already used NGINX as my reverse proxy. The installer runs Caddy on the host network and tries to bind ports 80/443, which conflicted with NGINX. Caddy failed to start.

To fix this, I removed the auto-launched containers and wrote my own `compose.yml` that only runs the PDS (no Caddy, no watchtower). I mapped it to port `6010` on the host instead of `3000`.

Here’s what my compose.yml file looks like:
```yaml
version: '3.9'
services:
  pds:
    container_name: pds
    image: ghcr.io/bluesky-social/pds:0.4
    restart: unless-stopped
    volumes:
      - type: bind
        source: /opt/pds
        target: /pds
    ports:
      - '6010:3000'
    env_file:
      - /opt/pds/pds.env
```
By default, the script stores data at `/pds` on the host. This is all your PDS state. You should back this up.

I moved it to `/opt/pds` to match my layout and updated the volume mount. You can keep `/pds` if you prefer.

Note: If you move the host directory, do not change paths in `pds.env`. Inside the container it is always mounted at `/pds`.

I keep `compose.yml` in a `bsky` directory and run `docker compose up -d` to start it.
## NGINX Reverse Proxy

With the container running on port 6010, I configured NGINX to expose it to the internet.

I also host other sites on `skiddle.id`, so I couldn’t let the PDS take over the whole domain (the installer expects this).

Luckily, the PDS only needs a few paths. I kept my other sites and proxied only those paths to the PDS.

The paths that it needs are:

- `/xrpc/`: This is the main path under which all of the APIs/RPCs that the PDS exposes are available
- `/.well-known/atproto-did`: This is used to verify domain ownership in addition to DNS methods
  - Note: You may not need this if you use DNS to verify your domain ownership. However, I was having issues with verifying my domain/handle and set up both methods.

Here’s the NGINX config I used to expose those paths to the PDS:
```nginx
server {
    server_name pds.skiddle.id www.pds.skiddle.id;

    # ... rest of your server config ...

    location /xrpc {
        proxy_pass http://localhost:6010;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header Host            $host;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }

    location /.well-known/atproto-did {
        proxy_pass http://localhost:6010/.well-known/atproto-did;
        proxy_set_header Host            $host;
    }

    # ...
}
```
It’s straightforward, but a few details matter:

For `/.well-known/atproto-did`, you must forward the `Host` header. The PDS uses it to decide which DID to return.

I added this to `pds.env` to help with the `/.well-known/atproto-did` route:
```sh
PDS_SERVICE_HANDLE_DOMAINS=.pds.skiddle.id
```

(replace with your domain)

This might not be required, but if the route still fails (even with the `Host` header), try it.

Also set the `Upgrade` and `Connection` headers to support websockets. You may need extra proxy config to enable websockets.

At first I thought websockets were optional (like for real-time notifications).

But they are required for a PDS to work and talk to the Bluesky network.

To test, I used an online websocket tester with this URL (change to your domain): `wss://pds.skiddle.id/xrpc/com.atproto.sync.subscribeRepos?cursor=0`

If the proxy is correct, the tester will report a successful connection.

## DNS for Handle Verification

To use `skiddle.id` as your handle while serving the PDS at `pds.skiddle.id`, set DNS like this:

```dns
# PDS service endpoint
pds.skiddle.id.   300  IN  A      <YOUR_SERVER_IPV4>
# pds.skiddle.id. 300  IN  AAAA   <YOUR_SERVER_IPV6>   ; optional

# Handle verification (root domain)
_atproto.skiddle.id. 300 IN TXT   "did=did:plc:YOUR_PLC_DID"
```

Notes:
- Replace placeholders with your values. The `_atproto` TXT proves you control `skiddle.id`.
- You can also serve `/.well-known/atproto-did` via your proxy. Either DNS or well-known works.
- If you use a CDN or extra proxy, enable websockets and forward headers.
## Issues with Top-Level Domain Handles

With the proxy working, I created an account using `pdsadmin` (as in the official docs).

I could log in at bsky.app and saw traffic hitting my PDS. But my account had issues.

My profile showed “Invalid Handle ⚠️”. I had already set DNS for verification, but it still failed—even after switching between root and subdomain handles.

The cause was broken websocket proxying. After fixing websockets, I ran this to request a re-crawl:

```sh
pdsadmin request-crawl bsky.network
```

Within a minute, the “Invalid Handle” was gone and my handle was set to `skiddle.id`.
### Other Debugging Resources

If you have handle or domain verification issues, this tool helps:

There is an official debugging app to help check your domain verification and handle status:

[https://bsky-debug.app/](https://bsky-debug.app/)


## Top-Level Domain Handle Issues

Right now, you can’t use a top-level domain as the handle for the first account on a fresh self-hosted PDS.

This is a limitation of the current setup, not of AT Protocol.

Workaround: start with a subdomain (I used `jeff.skiddle.id`), then switch to the root domain (`skiddle.id`) in bsky.app once websockets and DNS are correct.
## SMTP Server

I also set up SMTP so email verification would work. This might not be required for a single-user PDS, but it clears the UI warning.

I first tried Resend (as suggested in the docs), but I already have MX records set and didn’t want to change them to verify with Resend.

I use Google Workspace, so I used Google’s SMTP relay. It worked fine.

I added these lines to `pds.env`:
```sh
PDS_EMAIL_SMTP_URL=smtp://smtp-relay.gmail.com/
PDS_EMAIL_FROM_ADDRESS=jeff@skiddle.id
```

After that, verification emails delivered successfully.
## Conclusion

That’s it. After setup, the PDS has been stable and works normally with Bluesky.

I hope this helps you avoid the same issues.

Feel free to tag or message me on Bluesky [@skiddle.blue](https://bsky.app/profile/did:plc:xaj6j6abvpgwps5v6k4t6clv) if you have any questions as well :)