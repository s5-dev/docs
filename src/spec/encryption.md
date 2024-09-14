# Encryption

> This specification is still a work-in-progress draft. If you spot any issues or have suggestions on how it could be improved, please create an issue here: https://github.com/s5-dev/docs/issues

S5 supports different types of encryption, used in the <file-system.md> and other parts of the spec.

It ensures secure end-to-end-encryption when users need it, like in their private file system or when sending messages over a stream.

## Supported algorithms

ID | Cipher
- | -
**2** | AES-GCM (AES-256-GCM)
**4** | XChaCha20-Poly1305

S5 implementations must support both `AES-GCM` and `XChaCha20-Poly1305` for immutable and mutable encrypted blobs.

## Immutable Encrypted Blobs

Immutable Encrypted Blobs are used for file versions in the S5 <file-system.md>.

They have some parameters:
- **cipher**: The cipher used by the blob
- **chunk size**: The chunk size used for encrypting the blob (must be a power of 2 and larger than 1024 bytes). By default, S5 uses 256 KiB chunks.
- **encrypted blob hash**: The BLAKE3 hash of the encrypted blob
- **key**: The encryption key used for the blob
- **padding**: How much padding is added to the end of the blob (before encryption).
- **plaintext blob hash**: The BLAKE3 hash of the blob, before encryption and padding
- **plaintext blob size**: The size of the blob, before encryption and padding

The implementation itself is pretty simple: Every chunk is encrypted with the key, using the chunk index in the blob as a nonce (little-endian encoded). This is secure, because a new encryption key is randomly generated for every blob (and thus file version).

There's no spec (or need) for a way to encode immutable blobs as CIDs yet, because they only appear in the S5 file system file version objects, which are specifically designed to support the metadata parameters needed for encrypted blobs.

## Mutable Encryption

Mutable Encryption can be used when a key needs to be re-used for multiple revisions of a metadata file (like a directory) or messages (in a stream).

Payloads are currently limited to 1 MiB in size and padding is used by default to obfuscate the true data size.

XChaCha20-Poly1305 is the default cipher for mutable encrypted blobs.

### Directory CIDs with an encryption key

If you share a directory with someone using a CID, it usually consists of the bytes `5d ed 32_BYTE_PUBKEY`, which indicates that it points to a S5 Directory metadata file (`0x5d`, see <file-system.md>), that it's mutable by using an ed25519 pubkey (`0xed`) that points to a registry entry.

But if the directory is encrypted, you also need a key. In that case, a CID is encoded like this:

```
5d 5e e4 32_BYTE_ENCRYPTION_KEY ed 32_BYTE_PUBKEY
```

```
0x5d   S5 Directory CID magic byte
0x5e   Directory uses mutable encryption
0xe4   Mutable encryption uses the XChaCha20-Poly1305 cipher (see "Supported algorithms")
       In the case of AES-256-GCM, it would instead be 0xe2

32_BYTE_ENCRYPTION_KEY   This is the 32-byte key needed to decrypt the directory metadata with XChaCha20-Poly1305

ed 32_BYTE_PUBKEY   The ed25519 pubkey pointing to the registry entry is the same as with non-encrypted directories.
```

### Encryption (in Dart)

TODO: Does it make sense to use a magic byte prefix for encrypted mutable files, or would that be a risk?

```dart
const encryptionNonceLength = 24;
const encryptionOverheadLength = 16;

Future<Uint8List> encryptMutableBytes(
  Uint8List data,
  Uint8List key, {
  required CryptoImplementation crypto,
}) async {
  final lengthInBytes = encodeEndian(data.length, 4); // 4 bytes

  final totalOverhead =
      encryptionOverheadLength + lengthInBytes.length + encryptionNonceLength;

  final finalSize =
      padFileSizeDefault(data.length + totalOverhead) - totalOverhead;

  // Prepend the data size and append the padding bytes
  data = Uint8List.fromList(
    lengthInBytes + data + Uint8List(finalSize - data.length),
  );

  // Generate a random nonce.
  final nonce = crypto.generateRandomBytes(encryptionNonceLength);

  // Encrypt the data.
  final encryptedBytes = await crypto.encryptXChaCha20Poly1305(
    key: key,
    plaintext: data,
    nonce: nonce,
  );

  // Prepend the nonce to the final data.
  return Uint8List.fromList(nonce + encryptedBytes);
}
```

### Decryption (in Dart)

```dart
const encryptionKeyLength = 32;

Future<Uint8List> decryptMutableBytes(
  Uint8List data,
  Uint8List key, {
  required CryptoImplementation crypto,
}) async {
  if (key.length != encryptionKeyLength) {
    throw 'wrong encryptionKeyLength (${key.length} != $encryptionKeyLength)';
  }

  // Validate that the size of the data corresponds to a padded block.
  if (!checkPaddedBlock(data.length)) {
    throw "Expected parameter 'data' to be padded encrypted data, length was '${data.length}', nearest padded block is '${padFileSizeDefault(data.length)}'";
  }

  // Extract the nonce.
  final nonce = data.sublist(0, encryptionNonceLength);

  final decryptedBytes = await crypto.decryptXChaCha20Poly1305(
    key: key,
    nonce: nonce,
    ciphertext: data.sublist(encryptionNonceLength),
  );

  final lengthBytes = decryptedBytes.sublist(0, 4);
  final length = decodeEndian(lengthBytes);

  return decryptedBytes.sublist(4, length + 4);
}
```

## Padding

S5 uses "pad blocks" for padding, with the algorithm taken from the Sia Skynet project.

```dart
/// MIT License
/// Copyright (c) 2020 Nebulous

/// To prevent analysis that can occur by looking at the sizes of files, all
/// encrypted files will be padded to the nearest "pad block" (after encryption).
/// A pad block is minimally 4 kib in size, is always a power of 2, and is always
/// at least 5% of the size of the file.
///
/// For example, a 1 kib encrypted file would be padded to 4 kib, a 5 kib file
/// would be padded to 8 kib, and a 105 kib file would be padded to 112 kib.
/// Below is a short table of valid file sizes:
///
/// ```
///   4 KiB      8 KiB     12 KiB     16 KiB     20 KiB
///  24 KiB     28 KiB     32 KiB     36 KiB     40 KiB
///  44 KiB     48 KiB     52 KiB     56 KiB     60 KiB
///  64 KiB     68 KiB     72 KiB     76 KiB     80 KiB
///
///  88 KiB     96 KiB    104 KiB    112 KiB    120 KiB
/// 128 KiB    136 KiB    144 KiB    152 KiB    160 KiB
///
/// 176 KiB    192 Kib    208 KiB    224 KiB    240 KiB
/// 256 KiB    272 KiB    288 KiB    304 KiB    320 KiB
///
/// 352 KiB    ... etc
/// ```
///
/// Note that the first 20 valid sizes are all a multiple of 4 KiB, the next 10
/// are a multiple of 8 KiB, and each 10 after that the multiple doubles. We use
/// this method of padding files to prevent an adversary from guessing the
/// contents or structure of the file based on its size.
///
/// @param initialSize - The size of the file.
/// @returns - The final size, padded to a pad block.

int padFileSizeDefault(int initialSize) {
  final kib = 1 << 10;
  // Only iterate to 53 (the maximum safe power of 2).
  for (var n = 0; n < 53; n++) {
    if (initialSize <= (1 << n) * 80 * kib) {
      final paddingBlock = (1 << n) * 4 * kib;
      var finalSize = initialSize;
      if (finalSize % paddingBlock != 0) {
        finalSize = initialSize - (initialSize % paddingBlock) + paddingBlock;
      }
      return finalSize;
    }
  }
  // Prevent overflow.
  throw "Could not pad file size, overflow detected.";
}

bool checkPaddedBlock(int size) {
  final kib = 1024;
  // Only iterate to 53 (the maximum safe power of 2).
  for (int n = 0; n < 53; n++) {
    if (size <= (1 << n) * 80 * kib) {
      final paddingBlock = (1 << n) * 4 * kib;
      return size % paddingBlock == 0;
    }
  }
  throw "Could not check padded file size, overflow detected.";
}
```