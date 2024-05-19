# Blobs

As explained in [/concepts/content-addressed-data.html](/concepts/content-addressed-data.md), S5 uses the concept of content-addressing for all data and files, so any blob of bytes.

IPFS introduced the concept of Content Identifiers (CIDs), to have a standardized and future-proof way to refer to content-addressed data. Unfortunately, ["IPFS CIDs are not file hashes"](https://docs.ipfs.tech/concepts/content-addressing/#cids-are-not-file-hashes) because they split files up in a lot of small chunks, to make verified streaming of file slices possible without needing to download the entire file first. As a result, these files will never match their "true" hash, like when running `sha256sum`.

Fortunately, there has been some innovation in the space of cryptographic hash functions recently! Namely [BLAKE3](https://blake3.io/), which is based on the more well-known BLAKE2 hash function. Apart from being very fast and secure, its most unique feature is that its internal structure is already a Merkle tree. So instead of having to build a Merkle tree yourself (that's what IPFS does, CIDs point to the hash of a Merkle tree), BLAKE3 already takes care of that. As a result, CIDs using BLAKE3 are always consistent (for example with running `b3sum` on your local machine) and work with files of pretty much any size, while still supporting verified streaming (at any chunk size, down to `1024` bytes). So there's no longer a need to split up files bigger than 1 MiB in multiple chunks.

You can also check out the documentation of Iroh, another content-addressed data system, which explains this in a more in-depth way: <a href="https://iroh.computer/docs/layers/blobs" target="_blank">https://iroh.computer/docs/layers/blobs</a>


## Cool, but why yet another new CID format?

With bigger blobs and no extra metadata (due to the unaltered input bytes always being the source of a CID hash, so no longer using something like UnixFS), there's a need for knowing the file size of a Blob CID. So S5 continues to use (and be fully compatible) with BLAKE3 IPFS CIDs (and limited compatibility with other hash functions like sha256) when the blob size doesn't matter, but for use cases where it does, it introduces a new CID format.

Other protocols like the AT Protocol (used in Bluesky) solve this by using JSON maps for referencing blobs which contain both the IPFS CID and the blob size in an extra field. But I feel like there's value in having a compact format for representing an immutable sequence of bytes including its hash, so here we are.

> IPFS CIDs can be easily converted to S5 Blob CIDs if you know their file/blob size in bytes. If the IPFS CID is using the "raw binary IPLD codec", this operation is lossless. S5 Blob CIDs can always be converted to IPFS CIDs, but if the blob is bigger than 1 MiB it likely won't work with most IPFS implementations. S5 Blob CIDs can be losslessly converted to Iroh-compatible CIDs and back (assuming you keep the blob size somewhere or do a BLAKE3 size proof using Iroh)

## The S5 Blob CID format

S5 Blob CIDs always start with two magic bytes.

The first one is `0x5b` and indicates that the CID is a S**5** **b**lob CID.

The second one is `0x82` and indicates that it is a plaintext blob. `0x83` is reserved for encrypted blobs. (spec for them is still WIP)

Byte | Meaning
- | -
0x5b | S5 Blob CID magic byte
0x82 | S5 Blob Type Plaintext (Unencrypted, just a simple blob)

> As a nice side effect of picking exactly these two bytes, all S5 Blob CIDs start with with the string "blob" when encoded as base32 (multibase). All S5 CID magic bytes are picked carefully to not collide with any existing magic bytes on the <a href="https://github.com/multiformats/multicodec" target="_blank">https://github.com/multiformats/multicodec</a> table

After the two magic bytes, a single byte indicates which cryptographic hash function was used to derive a hash from the blob bytes. All S5 implementations should use `0x1e` (for BLAKE3), but SHA256 is also supported for compatibility reasons. SHA256 should only be used for small blobs imported from other systems, like IPFS or the AT Protocol.

Byte | Meaning
- | -
0x1e | multihash blake3
0x12 | multihash sha2-256

After the single multihash indicator byte, the 32 hash bytes follow. (S5 Blob CIDs always use the default hash output length, 32 bytes, for both blake3 and sha2-256. If the need for a different output length emerges in the future, a new possible value for the hash byte could be added)

Finally, the size (in bytes) of the blob is encoded as a little-endian byte array, trailing zero bytes are trimmed, and the remaining bytes appended to the CID bytes. Doing that could look like this in Rust (you can see a full example of calculating a CID in Rust at the bottom of this page):

```rust
let blob_size: u64 = 100_000_000_000; // 100 GB (you would usually just use .len() or something)
let mut cid_size_bytes = blob_size.to_le_bytes().to_vec();
if let Some(pos) = cid_size_bytes.iter().rposition(|&x| x != 0) {
    cid_size_bytes.truncate(pos + 1);
}
println!("{:?}", cid_size_bytes);
```

If we put all of this together, this is how the S5 Blob CID of the string `Hello, world!` in hex representation would look like: 

```hex
5b 82 12 ede5c0b10f2ec4979c69b52f61e42ff5b413519ce09be0f14d098dcfe5f6f98d 0d
PREFIX   BLAKE3 HASH (from b3sum)                                         SIZE
```

So the length of a S5 Blob CID depends on the filesize:
- Files with a size of less than 256 bytes have a 36-byte CID
- Files with a size of less than 64 KiB bytes have a 37-byte CID
- Files with a size of less than 16 MiB bytes have a 38-byte CID
- Files with a size of less than 4 GiB bytes have a 39-byte CID
- ...
- Files with a size of less than 16384 PiB have a 43-byte CID

> S5 Blob CIDs DO NOT contain a blob or file's media type, encoding or purpose. The reason for this is that it would no longer result in fully deterministic CIDs, because for example the media type could be interpreted differently by different applications or libraries.

## Encoding the S5 Blob CID bytes to a human-readable string

S5 uses the <a href="https://github.com/multiformats/multibase" target="_blank">multibase standard</a> for encoding CIDs, just like IPFS, Iroh and the AT Protocol.

S5 implementations MUST support the following self-identifying base encodings:

```csv
character,  encoding,           description
f,          base16,             Hexadecimal (lowercase)
b,          base32,             RFC4648 case-insensitive - no padding
z,          base58btc,          Base58 Bitcoin
u,          base64url,          RFC4648 no padding
```

For the string `Hello, world!`, these would be the S5 Blob CIDs in different encodings:

```
base16:    f5b8212ede5c0b10f2ec4979c69b52f61e42ff5b413519ce09be0f14d098dcfe5f6f98d0d
base32:    blobbf3pfycyq6lwes6ogtnjpmhsc75nucnizzye34dyu2cmnz7s7n6i
base58:    z34yzrj3Qqm7uAbDFe9aFjH9VD3GuiCZsRrJ4HS7HqYT3LqW
base64url: uW4IS7eXAsQ8uxJecabUvYeQv9bQTUZzgm-DxTQmNz-X2-Q
```

## Calculating the S5 Blob CID of a file using standard command line utils

Step 1: Calculate the BLAKE3 hash of your file (might need to install `b3sum`). You could also use `sha256sum` instead (and then put `0x12` as the hash prefix in step 3)

```bash
b3sum file.mp4
```

Step 2: Encode the size of your file in little-endian hex encoding

```bash
wc -c file.mp4 | cut -d' ' -f1 | tr -d '\n' | xargs -0 printf "%016x" | tac -rs .. | sed --expression='s/[00]*$/\n/'
```

Step 3: Add the multibase prefix and magic bytes

Characters | Purpose
- | -
f  | multibase prefix for Hexadecimal (lowercase)
5b | S5 Blob CID magic byte
82 | S5 Blob Type Plaintext (Unencrypted, just a simple blob)
1e | multihash blake3

Now, put it all together (the zeros will be your hash and the `654321` suffix your file size):

`f5b821e + BLAKE3_HASH + SIZE_BYTES = f5b821e0000000000000000000000000000000000000000000000000000000000000000654321`

That's it, you can now use that CID to trustlessly stream exactly that file from the S5 Network!

## Calculating a S5 Blob CID in Rust (using only top 100 crates)

```rust,edition2021
use data_encoding::BASE32_NOPAD; // 2.5.0;
use sha2::{Digest, Sha256}; // 0.10.8

fn main() {
    let blob = b"Hello, world!";
    
    let cid_prefix_bytes = vec![
        0x5b, // S5 Blob CID magic byte
        0x82, // S5 Blob Type Plaintext (Unencrypted, just a simple blob)
        0x12, // multihash sha2-256
    ];
    
    let sha256_hash_bytes = Sha256::digest(blob).to_vec();
    
    let blob_size = blob.len() as u64;
    let mut cid_size_bytes = blob_size.to_le_bytes().to_vec();
    if let Some(pos) = cid_size_bytes.iter().rposition(|&x| x != 0) {
        cid_size_bytes.truncate(pos + 1);
    }
    
    let cid_bytes = [cid_prefix_bytes, sha256_hash_bytes, cid_size_bytes].concat();
    
    println!("b{}", BASE32_NOPAD.encode(&cid_bytes).to_lowercase());
}
```
