# File System (FS5)

> This specification is still a work-in-progress draft. If you spot any issues or have suggestions on how it could be improved, please create an issue here: https://github.com/s5-dev/docs/issues

The S5 file system (FS5) is a decentralized, end-to-end-encrypted (if needed), content-addressed, versioned file system built using all primitives explained in the other S5 specifications.

> This specification is not fully complete yet, it will be updated shortly together with the Dart reference implementation

## Directory CIDs

S5 supports sharing and referencing directories using CIDs. They are currently either 34 bytes long (for non-encrypted directories) and longer for encrypted directories (see <encryption.md>).

The first byte is always `0x5d`.

The second byte is either `0x1e` (blake3) for immutable directories, or `0xed` (ed25519 pubkey for the registry) if you need a mutable pointer.

Finally, the last 32 bytes are either the blake3 hash or the ed25519 pubkey, depending on the previous byte.

The CID bytes are then encoded using multibase, see <blobs.md> for details on that.

When listing a directory, you first check if it's immutable or not. If yes, you just download the metadata blob using the blake3 hash from the network. If not, you first resolve the ed25519 pubkey to a blake3 hash using the <registry.md> and then download the metadata blob.

S5 directories can also be end-to-end-encrypted, as described in <encryption.md>.

## FS5 Directory Schema

The schema is defined in Rust. See the documentation of the `msgpack_schema` crate for details on what the annotations mean.

```rust
use msgpack_schema::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
#[untagged]
struct FS5Directory {
    file_signature: String, // constant "FS5.io" (magic bytes)
    header: FS5DirectoryHeader,
    directories: BTreeMap<String, DirectoryReference>,
    files: BTreeMap<String, FileReference>,
}
```

### Header

```rust
#[derive(Deserialize, Serialize)]
struct FS5DirectoryHeader {
    // 1: info string
    // 2: created_ts

    #[tag = 4] // TODO check
    #[optional]
    previous_version: Option<Vec<u8>>,

    #[tag = 6] // TODO check
    #[optional]
    share_state: Option<FS5DirectoryHeaderShareState>,

    // TODO optional HAMT sharding or B-Tree info
    // TODO 0x07: registry entry pointing to this
    // TODO 0x08: stream pointing to this? for incremental changes
}

#[derive(Deserialize, Serialize)]
struct FS5DirectoryHeaderShareState {
    #[tag = 1]
    #[optional]
    ro_shared_ts: Option<u64>,

    #[tag = 2]
    #[optional]
    rw_shared_ts: Option<u64>,
}

```

### Directory Reference

```rust
#[derive(Deserialize, Serialize)]
struct DirectoryReference {

    #[tag = 1]
    link: Vec<u8>, // always 32 bytes
    // TODO What if we want to encrypt the link?

    //#[tag = 2]
    //link_type: u8, // either 0xed pubkey or 0x1e static

    #[tag = 3]
    created_ts: u64,

    #[tag = 4]
    name: String,

    // ====================================

    #[tag = 4]
    // keys: DirectoryKeySet,
    // {key_id: {how_to_get_there: value}}
    keys: DirectoryKeySet,

    // ====================================

    #[tag = 14] // 0x0e
    #[optional]
    ext: Option<BTreeMap<String, msgpack_value::Value>>,
}

```

### File Reference

```rust
#[derive(Deserialize, Serialize)]
struct FileReference {
    #[tag = 1] 
    file: FileVersion,

    #[tag = 3]
    created_ts: u64,

    #[tag = 14] // 0x0e
    #[optional]
    ext: Option<BTreeMap<String, msgpack_value::Value>>,
}

#[derive(Deserialize, Serialize)]
struct FileVersion {
    #[tag = 1]
    size: u64,
    #[tag = 3]
    created_ts: u64,

    // TODO is_deleted: Option<bool>,

    // TODO Encryption paramenters (maybe support multiple blobs) 

    #[tag = 0x1e]
    hash_blake3: Vec<u8>,

    #[tag = 14] // 0x0e
    #[optional]
    ext: Option<BTreeMap<String, msgpack_value::Value>>,
}
```

## FS5 URI Format

`fs5://DIRECTORYCID/path/to/file.txt`

Example (long because hex-encoded): `fs5://f5ded0000000000000000000000000000000000000000000000000000000000000000/video.mkv`