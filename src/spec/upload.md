# Upload

S5 Nodes provide two HTTP-based APIs for uploading files, depending on the file size and if you need resumable uploads.

## The simple one for small files

Small files can be uploaded with a single HTTP POST request, using the default file upload form field:

`curl -X POST "https://S5_NODE_URL/s5/upload" -F "file=@example.txt"`

Please note that non-localhost (or local network) S5 Nodes reachable on the Internet usually require authentication for uploading files, see [accounts.md](accounts.md) for details.

`curl -X POST "https://S5_NODE_URL/s5/upload?auth_token=AUTH_TOKEN_HERE" -F "file=@example.txt"`

## TUS for larger files

For larger (resumable) file uploads, S5 uses the https://tus.io/ protocol.

Before starting an upload, you need to calculate the blake3 hash of the file locally. When creating the tus upload using the initial `POST` request, you need to pass the hash as metadata. With most tus libraries, you would pass the hash like this:

```json
{
    "hash": "BASE64URL_ENCODE(0x1e + HASH_HERE)"
}
```

It's available on the `/s5/upload/tus` endpoint and also usually requires authentication.
