# S5 Config

You can edit the `config.toml` file to configure your S5 node. You can apply changes with `docker container restart s5-node`

This page describes the available sections in the config.

### keypair

The `seed` is generated on first start, you should keep it private. It's used for signing messages on the network.

### http.api

`domain`: Configure this value to match the domain you are using to access your node. If you for example configured your domain `example.com` to be reverse-proxied to your S5 Node Docker container using Caddy, nginx or others, you should set this to `example.com`

`port`: On which port the HTTP API should bind to and be available (you should usually keep this the default)

### store

Check out the [Stores documentation](/stores/index.html) for configuring different object stores.

### accounts

You can enable the accounts system by adding this part to your config:

```toml
[accounts]
enabled = true
[accounts.database]
path = "/db/accounts"
```

Registrations are disabled by default, you can enable them by adding this part:

```toml
[accounts]
alwaysAllowedScopes = [
    'account/login',
    'account/register',
    's5/registry/read',
    's5/metadata',
    's5/debug/storage_locations',
    's5/debug/download_urls',
    's5/blob/redirect',
]
```

## Advanced

### cache

Configure a custom cache path with `path`, you likely don't need this if you are using Docker.

### database

Configure a custom database path, you likely don't need this if you are using Docker.

### p2p.peers

List of initial peers used for connecting to the p2p network.