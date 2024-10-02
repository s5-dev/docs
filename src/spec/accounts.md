# Accounts on Storage Nodes

Many use cases of S5 require either storing more data that can fit on a user's local device or users might simply want to use a hosted provider to store their data.

Fortunately, S5 makes this mostly trustless by using integrity verification (blake3 hashes) for all file uploads and downloads, so your storage provider can do very little to tamper with your files.

That said, you can have a higher risk of losing access to your data when using an external storage provider. They could for example simply delete your files, revoke your access, be hacked or fall victim to a datacenter fire. Even when using a S5 Node on your local NAS as a storage provider, some of these risks still apply.

So to prevent losing access to your data in these cases, S5 tries to make it as easy as possible to keep your data mirrored on multiple independent storage providers. One part of this is using content-addressing and a decentralized network, which makes it possible for you to quickly pin files on additional storage services without needing to re-upload them (because the storage providers can communicate with each other directly via the S5 Network).

S5 provides an anonymous accounts system, which makes it easy to manage and authenticate on many different storage providers (S5 Nodes) at once with minimal metadata. A storage provider only gets a randomly generated pubkey from you when logging in, which is different for every provider you sign up to. Some providers might require additional data (like an email address) to sign up, but that's not part of the S5 protocols.

## Account Tiers

>> This part is not written yet.

## Relevant HTTP APIs

See <https://github.com/s5-dev/S5/blob/main/lib/service/accounts.dart> for the current reference implementation, more detailed API documentation will be added to this spec in the future.

### `/s5/account/register`

This endpoint (first GET for the challenge, then POST with signed payload to register) can be used to register a new account using a specific ed25519 public key. How that key is generated is up to the implementation and use case, but for apps using the S5 Identity system it's described in [identity.md](identity.md).

### `/s5/account/login`

This endpoint (first GET for the challenge, then POST with signed payload to login) can be used to generate a new authentication token for a specific account. The token can then be used to upload blobs, list pinned blobs, pin new blobs, read stats and delete blobs.

### Authentication

Calls to the S5 APIs below can be authenticated by either passing the token as a header `Authorization: Bearer AUTH_TOKEN_HERE` or as a query parameter for simplicity `?auth_token=AUTH_TOKEN_HERE`

### `/s5/account/stats`

This endpoint requires authentication in the form of a token. It returns account details and statistics related to storage usage.




