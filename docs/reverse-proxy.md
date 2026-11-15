# Reverse proxy and TLS

The app itself speaks plain HTTP on port 8088. For anything beyond a single
trusted workstation, put a TLS reverse proxy in front of it, such as Nginx
Proxy Manager (NPM), nginx, Caddy or Traefik. The security reasoning is in
[SECURITY.md](../SECURITY.md#running-behind-tls-reverse-proxy).

## App settings

In `.env`:

```
TRUST_PROXY=1
WEB_BIND=127.0.0.1
```

- `WEB_BIND` is the host interface the container's port 8088 is published on.
  The default `127.0.0.1` keeps the app off the network, so only something on
  the same host (the proxy) can reach it. `0.0.0.0` publishes it on every
  interface; only do that if you must reach `:8088` directly.
- `TRUST_PROXY=1` turns on a small middleware (`TrustProxy` in
  `web/config.ru`). It reads `X-Forwarded-Proto` and `X-Forwarded-Port` from the
  proxy and uses them as the request's scheme and port, so OAuth redirect URIs
  and absolute URLs come out as `https://`. It also marks the session cookie
  `Secure`. If `X-Forwarded-Port` is missing, it uses 443 for `https` and 80 for
  `http`.
- **Only enable `TRUST_PROXY` when the app is reached solely through a proxy you
  control** that sets those headers. Otherwise a client could send its own
  `X-Forwarded-Proto`.
- Do not set `TRUST_PROXY=1` while you still browse over plain HTTP: the
  `Secure` cookie is not sent back over HTTP and login silently fails.

Restart after changing either value: `docker compose up -d`.

## Proxy settings

Whatever proxy you use:

- Forward to `127.0.0.1:8088` (or wherever `WEB_BIND` publishes it).
- **WebSockets must be enabled.** The graphical console (`/websockify/...`) and
  the text console (`/solws/...`) are WebSockets. Without WebSocket support the
  noVNC console connects and then drops.
- Set a long read timeout, for example `proxy_read_timeout 86400s;` in nginx,
  so an idle KVM or serial stream is not cut after the default 60 seconds.
- Pass the original `Host` header and set `X-Forwarded-Proto`.
- Use a real certificate (for example Let's Encrypt), force HTTPS, and enable
  HTTP/2 if you like.

Login callback URLs:

- GitHub OAuth: set the OAuth app's callback to
  `https://<your-domain>/auth/github/callback`.
- OIDC: register `https://<your-domain>/auth/oidc/callback` (strict match) with
  the identity provider, and set exactly the same value as `OIDC_REDIRECT_URI`.

Harden the proxy itself: keep its admin interface on a separate port or
network, and put an access list or IP allow-list in front of the app. That is
the proxy's job, not this app's.

### Nginx Proxy Manager

On the proxy host for the app:

- Forward hostname/IP `127.0.0.1`, port `8088`, scheme `http`.
- **Websockets Support: on.**
- Advanced tab: `proxy_read_timeout 86400s;`
- SSL tab: Let's Encrypt certificate, **Force SSL**, **HTTP/2 Support**.

### Plain nginx example

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 443 ssl http2;
    server_name bmc.example.com;

    ssl_certificate     /etc/ssl/certs/bmc.example.com.pem;
    ssl_certificate_key /etc/ssl/private/bmc.example.com.key;

    location / {
        proxy_pass http://127.0.0.1:8088;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade           $http_upgrade;
        proxy_set_header Connection        $connection_upgrade;
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }
}
```

## When the proxy runs in a container

`127.0.0.1` inside a container is that container, not the Docker host. If your
proxy (NPM, for example) runs in its own container on a bridge network, it
cannot reach a port published on the host's `127.0.0.1`. Options:

- Run the proxy with host networking, so its `127.0.0.1` is the host's.
- Or set `WEB_BIND` to an address the proxy container can reach, such as the
  host's Docker bridge address, and make sure that address is not reachable
  from the rest of the network.

## Rebuilds and open sessions

Rebuilding or restarting `bmc-web` drops any open KVM or text-console
WebSocket. The browser reconnects; no proxy setting is needed for that.
