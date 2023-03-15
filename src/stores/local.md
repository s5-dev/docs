# Local

Stores uploaded files on the local filesystem.

## Configuration

```toml
[store.local]
path = "/data" # If you are using the Docker container

[store.local.http]
bind = "127.0.0.1"
port = 8989
url = "http://localhost:8989"
```

By default, files will only be available on your local node.
To make it available on the entire network, you have to forward your port to be reachable from the internet and then update the `url` to the URL at which your computer is available from the internet.
