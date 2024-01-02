# Install the S5 node

Right now the only supported way to run a S5 node is using a **container runtime** like Docker or Podman.

You can install **Docker** on most operating systems using the instructions here: <https://docs.docker.com/engine/install/>

If you are on Linux you can use the convenience script: `curl -fsSL https://get.docker.com | sudo sh`

**Podman** is a popular alternative to Docker, but it might be harder to install on non-Linux system. You can find instructions for it here: <https://podman.io/getting-started/installation>

## Run S5 using Docker

Before running this command, you should change the paths `./s5/config` and `./s5/db` to a storage location of your choice.

```bash
docker run -d \
--name s5-node \
-p 127.0.0.1:5050:5050 \
-v ./s5/config:/config \
-v ./s5/db:/db \
--restart unless-stopped \
ghcr.io/s5-dev/node:latest
```

This will only bind your node to localhost, so you will need a reverse proxy like [Caddy](caddy.md) to access it from the internet.

If you instead want to expose the HTTP API port to the entire network, you can set `-p 5050:5050`

If something seems to not work correctly, you can view the logs with `docker logs -f s5-node`

### config path

This path will be used to generate and load the `config.toml` file, you will need to edit that file for configuring stores and other options.

### db path

This path is used for storing small key-value databases that hold state relevant for the network and node. Do not use a slow HDD for this.

### (optional) cache path

The cache stores large file uploads and different downloads/streams. You can use a custom cache location by adding `-v ./s5/cache:/cache` to your command.

### (optional) data path

If you are planning to store uploaded files on your local disk, you should prepare a directory for that and specify it with `-v ./s5/data:/data`

### Using Sia

If you want to use S5 with an instance of **renterd** running on the same server, you should add the `--network="host"` flag to grant S5 access to the renterd API.

### Stop the container

```sh
docker container stop s5-node
```

### Remove the container

```sh
docker container rm s5-node
```

## Alternative: Using docker-compose

Create a file called `docker-compose.yml` with this content:

```yml
version: '3'
services:
  s5-node:
    image: ghcr.io/s5-dev/node:latest
    volumes:
      - ./path/to/config:/config
    ports:
      - "5050:5050"
    restart: unless-stopped
```

Same configuration options as with normal Docker/Podman, run it with `docker-compose up -d`