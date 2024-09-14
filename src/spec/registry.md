# Registry

> This specification is still a work-in-progress draft. If you spot any issues or have suggestions on how it could be improved, please create an issue here: https://github.com/s5-dev/docs/issues

The registry is a distributed key-value store on the S5 Network and makes mutable data structures possible by being a pointer to immutable blobs. At the moment, all keys are **ed25519 public keys** and each entry must be signed by the corresponding private key. All keys on the network are prefixed with the `0xed` byte (indicating ed25519) to make future additions to the set of supported algorithms easy (especially quantum-safe ones). Each entry also has a 64-bit unsigned integer **revision number**. Nodes should drop any (valid) registry entries they receive if their revision number is lower than that of an entry for the same public key they already know about.

Registry entries should be stored for as long as possible by nodes receiving them. To make this easier, registry entries can hold a maximum of 48 bytes of data. While small, this is more than enough to store a Blob CID or simple hash reference. So it shouldn't put any limits on possible use cases.

> The code examples below are written in the Dart programming language. The final spec will use Rust ones.

The registry uses little-endian byte encoding for the `u64` revision.


## Registry Entry structure

```dart
class SignedRegistryEntry {
  /// public key with multicodec prefix (0xed for ed25519)
  final Uint8List pk;

  /// revision number of this entry, maximum is (256^8)-1
  final int revision; // must be an unsigned 64-bit integer.

  /// data stored in this entry, can have a maximum length of 48 bytes
  final Uint8List data;

  /// signature of this signed registry entry
  final Uint8List signature;
```

## Relevant Constants

```dart
const recordTypeRegistryEntry = 0x07;
```

## Serializing a registry entry (for wire transport)

```dart
Uint8List serialize() {
    return Uint8List.fromList([
        recordTypeRegistryEntry, // 0x07 (constant)
        ...pk,
        ...encodeEndian(revision, 8), // this uses little-endian encoding
        data.length, // a single byte for the data length
        ...data,
        ...signature,
    ]);
}
```

## Signing a registry entry

```dart
Future<SignedRegistryEntry> signRegistryEntry({
  required KeyPairEd25519 kp,
  required Uint8List data,
  required int revision,
  required CryptoImplementation crypto,
}) async {
  final list = Uint8List.fromList([
    recordTypeRegistryEntry, // 0x07 (constant)
    ...encodeEndian(revision, 8),
    data.length, // 1 byte
    ...data,
  ]);

  final signature = await crypto.signEd25519(
    kp: kp,
    message: list,
  );

  return SignedRegistryEntry(
    pk: kp.publicKey,
    revision: revision,
    data: data,
    signature: Uint8List.fromList(signature),
  );
}
```

## Verifying a registry entry

```dart
Future<bool> verifyRegistryEntry(
  SignedRegistryEntry sre, {
  required CryptoImplementation crypto,
}) {
  final list = Uint8List.fromList([
    recordTypeRegistryEntry, // 0x07 (constant)
    ...encodeEndian(sre.revision, 8),
    sre.data.length, // 1 byte
    ...sre.data,
  ]);
  return crypto.verifyEd25519(
    pk: sre.pk.sublist(1),
    message: list,
    signature: sre.signature,
  );
}
```
