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
trusting anyone except the person who gave you the hash. Other benefits include highly efficient caching (due to file blobs being immutable by default) and automatic deduplication of data.

## Verified streaming

To make verified streaming of large files possible, S5 uses the [Bao](https://github.com/oconnor663/bao) implementation for BLAKE3 verified streaming.
As mentioned earlier, BLAKE3 is a merkle tree on the inside - this makes it possible to verify the integrity of small parts of a file without having
to download and hash the entire file first.

By default, S5 stores some layers of the Bao hash tree next to every stored file that is larger than 256 KiB (same path, but `.obao` extension).
With the default layers, it's possible to verify chunks with a minimum size of 256 KiB from a file.
So if you're for example streaming a large video file, your player only needs to download the first 256 KiB of the file before being able to show the first frame.
The overhead of storing the tree is a bit less than 256 KiB per 1 GiB of stored data.

## CIDs (Content identifiers)

See [/spec/blobs.html](/spec/blobs.md) for up-to-date documentation on how S5 calculates CIDs.
 
## Media types

To make deduplication as efficient as possible, raw files on S5 do not contain any additional metadata like filenames or media types.
You can append a file extension to your CID to stream/share a single file with the correct content type, for example `zHnq5PTzaLbboBEvLzecUQQWSpyzuugykxfmxPv4P3ccDcGwnw.txt`.

For other use cases, you should use one of the [metadata formats](/metadata/index.html).