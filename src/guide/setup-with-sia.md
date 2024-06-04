# Setup With Sia S3

It is recommended to use the direct Sia API integration as documented in the [full renterd setup guide with S5](deploy-renterd.md). But S5 also supports the S3 endpoints.

First, you'll need a fully configured instance of **renterd** (the new Sia renter software) running somewhere. Here's a [great guide](https://blog.sia.tech/sia-innovate-and-integrate-christmas-2023-hackathon-9b7eb8ad5e0e) which shows you how to set one up easily on the Sia testnet.

Next, you need to set up a S5 Node using the instructions available [here](../install/index.html)

For configuring the S5 Node to use your Sia renter node, you will need to add this section to your `config.toml`:

```toml
[store.s3]
accessKey = "MY_ACCESS_KEY" # Replace this with the access key from your renterd.yml
bucket = "sfive" # Or just "default"
endpointUrl = "YOUR_S3_ENDPOINT_URL" # http://localhost:7070 if you followed the Sia renterd testnet guide
secretKey = "MY_SECRET_KEY" # Replace this with the secret key from your renterd.yml
```

And then restart the node with `docker container restart s5-node`

You might also want to enable the **accounts** system on your node if it's available on the internet or if you want to use it with Vup, see [config instructions](../install/config/index.html) for details.
