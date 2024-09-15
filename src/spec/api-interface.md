# API Interface

> This specification is still a work-in-progress draft. If you spot any issues or have suggestions on how it could be improved, please create an issue here: https://github.com/s5-dev/docs/issues

S5 contains a lot of features. To keep the system modular and make it easier to for example only implement some parts of the spec in a new programming language or environment, there are two standard API interfaces used by most systems. So if you implement these, you can just pass them to something like the FS.

## S5APIProvider

This is the base API that must be supported by all implementations. Some of them might add additional methods for extra features or convenience.

TODO Add purpose/context field to upload methods.

```dart
typedef OpenReadFunction = Stream<List<int>> Function([int? start, int? end]);

abstract class S5APIProvider {
  /// Blocks until the S5 API is initialized and ready to be used
  Future<void> ensureInitialized();

  /// Upload a small blob of bytes
  ///
  /// Returns the Raw CID of the uploaded raw file blob
  ///
  /// Max size is 10 MiB, use [uploadRawFile] for larger files
  Future<BlobCID> uploadBlob(Uint8List data);

  /// Upload a raw file
  ///
  /// Returns the Raw CID of the uploaded raw file blob
  ///
  /// Does not have a file size limit and can handle large files efficiently
  Future<BlobCID> uploadBlobWithStream({
    required Multihash hash,
    required int size,
    required OpenReadFunction openRead,
  });

  /// Downloads a full file blob to memory, you should only use this if they are smaller than 1 MB
  Future<Uint8List> downloadBlob(Multihash hash, {Route? route});

  /// Downloads a slice of a blob to memory, from `start` (inclusive) to `end` (exclusive)
  Future<Uint8List> downloadBlobSlice(
    Multihash hash, {
    required int start,
    required int end,
    Route? route,
  });

  Future<void> pinHash(Multihash hash);

  Future<void> unpinHash(Multihash hash);

  Future<SignedRegistryEntry?> registryGet(
    Uint8List pk, {
    Route? route,
  });
  Stream<SignedRegistryEntry> registryListen(
    Uint8List pk, {
    Route? route,
  });
  Future<void> registrySet(
    SignedRegistryEntry sre, {
    Route? route,
  });

  Stream<SignedStreamMessage> streamSubscribe(
    Uint8List pk, {
    int? afterTimestamp,
    int? beforeTimestamp,
    Route? route,
  });
  Future<void> streamPublish(
    SignedStreamMessage msg, {
    Route? route,
  });

  CryptoImplementation get crypto;
}
```

>> The `Route? route` argument is not used currently

## CryptoImplementation

```dart
abstract class CryptoImplementation {
  Uint8List generateSecureRandomBytes(int length);

  Future<Uint8List> hashBlake3(Uint8List input);

  Uint8List hashBlake3Sync(Uint8List input);

  Future<Uint8List> hashBlake3File({
    required int size,
    required OpenReadFunction openRead,
  });

  Future<bool> verifyEd25519({
    required Uint8List publicKey,
    required Uint8List message,
    required Uint8List signature,
  });

  Future<Uint8List> signEd25519({
    required KeyPairEd25519 keyPair,
    required Uint8List message,
  });

  Future<KeyPairEd25519> newKeyPairEd25519({
    required Uint8List seed,
  });

  Future<Uint8List> encryptXChaCha20Poly1305({
    required Uint8List key,
    required Uint8List nonce,
    required Uint8List plaintext,
  });

  Future<Uint8List> decryptXChaCha20Poly1305({
    required Uint8List key,
    required Uint8List nonce,
    required Uint8List ciphertext,
  });
}
```