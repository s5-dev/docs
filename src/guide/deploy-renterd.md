# Deploy a personal S5 Node with renterd storage on Debian

In this guide, you'll learn how to deploy a production-ready S5 Node backed by Sia renterd storage.

You can then use it with the Vup Cloud Storage app or just play with the S5 API directly to upload and manage files of any size!

## Requirements

- Domain Name (just a subdomain works too)
- Debian VPS (x86 or arm64) with 8+ GB of RAM and 128+ GB of free disk space (16+ GB of RAM are better for performance)
- Some SC (siacoin) for forming contracts on the network and renting storage

If you're looking for affordable providers with these specs, I found the new Netcup ARM Servers to be a pretty good choice (https://www.netcup.de/vserver/arm-server/)
- 7 EUR/month for 8 GB of RAM
- 12 EUR/month for 16 GB of RAM

## Install Sia `renterd`

Check out the official Sia docs for detailed instructions with screenshots: https://docs.sia.tech/renting/setting-up-renterd/linux/debian

Or just connect to your Debian VPS over SSH and copy-paste these commands:

```bash
sudo curl -fsSL https://linux.sia.tech/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/siafoundation.gpg
echo "deb [signed-by=/usr/share/keyrings/siafoundation.gpg] https://linux.sia.tech/debian $(. /etc/os-release && echo "$VERSION_CODENAME") main" | sudo tee /etc/apt/sources.list.d/siafoundation.list
sudo apt update
sudo apt install renterd
cd /var/lib/renterd
```

Run `renterd version` to verify it was installed correctly.

## Configure Sia renterd

Run `sudo renterd config` and follow the instructions. **Please choose a secure password for the renterd admin UI!** You can use `pwgen -s 42 1` to generate one.

Type **yes** when you're asked if you want to configure **S3 settings**.

Keep the `S3 Address` on the default setting. (You won't need it for this guide, but s3 support is very useful for many other potential use cases for your renterd node)

Finally, start renterd using `sudo systemctl start renterd`

Then you can re-connect to your VPS using `ssh -L localhost:9980:localhost:9980 IP_ADDRESS_OR_DOMAIN` to create a secure SSH tunnel to the renterd web UI. After connecting, you can open http://localhost:9980/ in your local web browser and authenticate with the previously set API password.

In the web UI, follow the step-for-step welcome guide and set everything up.

1. Configure the storage settings
2. Fund your wallet (https://docs.sia.tech/renting/transferring-siacoins)
3. Create a new bucket with the name `s5` on http://localhost:9980/files (top right button)
4. Wait for the chain to sync
5. Wait for storage contracts to form

## Install and setup Caddy (reverse proxy)

So first, you'll need to decide on two domains you want to use - one for the S5 node (important) and one for the download proxy to your renterd node (less important).

For example if you own the domain `example.com`, you could run the S5 Node on `s5.example.com` and the download proxy on `dl.example.com`.

You should then add DNS `A` records pointing to the IP address of your VPS for both of these subdomains. Of course `AAAA` records are nice too if you have IPv6.

Then install Caddy by following these instructions: https://caddyserver.com/docs/install#debian-ubuntu-raspbian

Then edit the Caddy config with `nano /etc/caddy/Caddyfile` to something like this:

You can generate the `TODO_PASTE_HERE` part by encoding your Sia renterd API password to base64 (input: `:APIPASSWORD`) on https://gchq.github.io/CyberChef/#recipe=To_Base64('A-Za-z0-9%2B/%3D')

```js
S5_API_DOMAIN {
  reverse_proxy 127.0.0.1:5050
}

DOWNLOAD_PROXY_DOMAIN {
  uri strip_suffix /
  header {
    Access-Control-Allow-Origin *
  }
  rewrite * /api/worker/objects/s5{path}?bucket=s5
  @download {
    method GET
  }
  reverse_proxy @download {
    to 127.0.0.1:9980
    header_up Authorization "Basic TODO_PASTE_HERE"
  }
}
```

Then restart Caddy with `systemctl restart caddy`

## Install and set up S5

First, install Podman using this command: `sudo apt-get -y install podman`

Create some needed directories: 

```sh
mkdir -p /s5/config
mkdir -p /s5/db
mkdir -p /tmp/s5
```

Then start a S5 node with these commands: (You might need to create the `/s5/` directories in that command first)

```sh
podman run -d \
--name s5-node \
--network="host" \
-v /s5/config:/config \
-v /s5/db:/db \
-v /tmp/s5:/cache \
--restart unless-stopped \
ghcr.io/s5-dev/node:latest
```

Edit `nano /s5/config/config.toml`

Add these config entries there:

```toml
[http.api]
domain = 'S5_API_DOMAIN'

[accounts]
enabled = true
[accounts.database]
path = "/db/accounts"

[store.sia]
workerApiUrl = "http://127.0.0.1:9980/api/worker"
bucket = "s5"
apiPassword = "APIPASSWORD"
downloadUrl = "https://DOWNLOAD_PROXY_DOMAIN"

```

Then run `podman restart s5-node` to restart the S5 Node.

You can visit https://S5_API_DOMAIN/s5/admin/app in your web browser to create and manage accounts manually. The API key for your node can be retrieved by running `journalctl | grep 'ADMIN API KEY'`

## Using your new S5 Node for Vup Storage

Edit `nano /s5/config/config.toml` and add

```toml
[accounts]
authTokensForAccountRegistration = ["INSERT_INVITE_CODE_OF_CHOICE_HERE"]
enabled = true
```

Then run `podman restart s5-node` to restart the S5 Node.

Now you can use the "Register on S5 Node" button in the Vup "Storage Service" settings, enter the domain of your node and the newly generated invite code and you should be good to go! You'll likely want to use more than 10 GB of storage, so just use the Admin Web UI to set a higher tier for your newly created account.
