# Caddy reverse proxy

Caddy is an easy to use reverse proxy with automatic HTTPS.

You can install it by following the instructions over at <https://caddyserver.com/docs/install>

You'll also need a domain name with `A` and `AAAA` records pointed to your server.

You should also make sure that your firewall doesn't block the ports `80` and `443`

## Configuration

With the default S5 port of `5050`, you can configure your `/etc/caddy/Caddyfile` like this:

```Caddyfile
YOUR.DOMAIN {
  reverse_proxy localhost:5050
}
```

On Debian and Ubuntu you can run `sudo systemctl restart caddy` to restart Caddy after editing the Caddyfile.

Don't forget to configure `http.api.domain` in your S5 `config.toml` after setting up a domain and reverse proxy!