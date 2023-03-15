# Content-addressed data

## Cryptographic hashes

Cryptographic hash functions are an algorithm that can map files or data of any size to a fixed-length hash value.

- They are deterministic, meaning the same input always results in the same hash
- It is infeasible to generate a message that yields a given hash value (i.e. to reverse the process that generated the given hash value)
- It is infeasible to find two different messages with the same hash value
- A small change to a message should change the hash value so extensively that it appears uncorrelated with the old hash value

## The BLAKE3 hash function

[BLAKE3](https://github.com/BLAKE3-team/BLAKE3) is a cryptographic hash function that is:

- **Much faster** than MD5, SHA-1, SHA-2, SHA-3, and BLAKE2.
- **Secure**, unlike MD5 and SHA-1. And secure against length extension,
  unlike SHA-2.
- **Highly parallelizable** across any number of threads and SIMD lanes,
  because it's a Merkle tree on the inside.
- Capable of **verified streaming** and **incremental updates**, again
  because it's a Merkle tree.
- A **PRF**, **MAC**, **KDF**, and **XOF**, as well as a regular hash.
- **One algorithm with no variants**, which is fast on x86-64 and also
  on smaller architectures.

## Content-addressing

Content-addressing means that instead of addressing data by their location (for example with protocols like HTTP/HTTPS), it's referenced
by their cryptographic hash. This makes it possible to make sure you actually received the correct data you are looking for without
trusting anyone except the person who gave you the hash. It also makes all files immutable by default.

## Verified streaming

To make verified streaming of large files possible, S5 uses the [Bao](https://github.com/oconnor663/bao) implementation for BLAKE3 verified streaming.
As mentioned earlier, BLAKE3 is a merkle tree on the inside - this makes it possible to verify the integrity of small parts of a file without having
to download and hash the entire file first.

By default, S5 appends some layers of the Bao hash tree to every stored file that is larger than 256 KiB.
With the default layers, it's possible to verify chunks with a minimum size of 256 KiB from a file.
So if you're for example streaming a large video file, your player only needs to download the first 256 KiB of the file before being able to show the first frame.
The overhead of storing the tree is a bit less than 256 KiB per 1 GiB of stored data.

## CIDs (Content identifiers)

Hash values produced by the BLAKE3 hash function have a size of 32 bytes, for example `c4d27f80613c2dfdc4d9d013b43c181576e21cf9c2616295646df00db09fbd95` (hex-encoded).

Instead of using this value directly, S5 prepends two additional bytes to reference raw files:

`0x26 cidTypeRaw`: This CID contains a raw file without any additional metadata

`0x1f mhashBlake3Default`: This CID contains a BLAKE3 hash of the file with the default 256-bit output size

You can find a list of all up-to-date magic bytes here: [lib5:constants.dart](https://github.com/s5-dev/lib5/blob/main/lib/src/constants.dart)

In addition to these two magic bytes, the size of the file (in bytes) is encoded with little-endian encoding and appended to the hash bytes.
For example a file with `18657` bytes, would be encoded like this:

```javascript
0x26 0x1f 0xc4d27f80613c2dfdc4d9d013b43c181576e21cf9c2616295646df00db09fbd95 0xe148
type hash blake3-256-hash                                                    filesize
```

So the length of a raw file CID depends on the filesize:
- Files with a size of less than 256 bytes have a 35-byte CID
- Files with a size of less than 64 KiB bytes have a 36-byte CID
- Files with a size of less than 16 MiB bytes have a 37-byte CID
- Files with a size of less than 4 GiB bytes have a 38-byte CID
- ...
- Files with a size of less than 16384 PiB have a 42-byte CID

## Encoding the CID bytes to a human-readable form

S5 uses the [multibase](https://github.com/multiformats/multibase) standard for encoding the CID bytes.
Basically the first character indicates how the bytes are encoded, here's a list of which ones are supported by S5:
```
base32,            b,    rfc4648 case-insensitive - no padding
base58btc,         z,    base58 bitcoin
base64url,         u,    rfc4648 no padding
```

By default, `base58btc` with the `z` prefix is used for newly uploaded files because it's short and easy to copy.

So the CID from the example earlier would be encoded like this:
```
base58btc: zHnq5PTzaLbboBEvLzecUQQWSpyzuugykxfmxPv4P3ccDcGwnw
base32:    beyp4jut7qbqtylp5ytm5ae5uhqmbk5xcdt44eylcsvsg34anwcp33fpbja
base64url: uJh_E0n-AYTwt_cTZ0BO0PBgVduIc-cJhYpVkbfANsJ-9leFI
```
 
## Media types

To make deduplication as efficient as possible, raw files on S5 do not contain any additional metadata like filenames or media types.
You can append a file extension to your CID to stream/share a single file with the correct content type, for example `zHnq5PTzaLbboBEvLzecUQQWSpyzuugykxfmxPv4P3ccDcGwnw.txt`.

For other use cases, you should use one of the [metadata formats](/metadata/index.html).