# Sia Network

[The Sia network](https://sia.tech/) provides decentralized and redundant data storage.

You will need a fully configured local instance of **renterd**: https://github.com/SiaFoundation/renterd

Warning: Both renterd and this integration are still experimental. Please report any bugs you encounter.

## Configuration

```toml
[store.sia]
workerApiUrl = "http://localhost:9980/api/worker"
apiPassword = "test"
downloadUrl = "https://dl.YOUR.DOMAIN"
```

## Using Caddy as a reverse proxy for Sia downloads

This configuration requires a version of Caddy with <https://github.com/caddyserver/cache-handler>, if you don't want to cache Sia downloads you can remove the first 4 lines and the **cache** directive.

`/etc/caddy/Caddyfile`:

```Caddyfile
{
    order cache before rewrite
    cache
}

dl.YOUR.DOMAIN {
  uri strip_suffix /

  header {
    Access-Control-Allow-Origin *
  }

  cache {
    stale 6h
    ttl 24h
    default_cache_control "public, max-age=86400"
    nuts {
      path /tmp/nuts
    }
  }

  rewrite * /api/worker/objects/1{path}

  reverse_proxy {
    to localhost:9980
    header_up Authorization "Basic OnRlc3Q=" # Change this to match your renterd API key
  }
}
```