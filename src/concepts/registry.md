# Registry

The S5 registry is a decentralized key-value store. A registry entry looks like this:

```typescript
class SignedRegistryEntry {
  // public key with multicodec prefix
  pk: Uint8Array;

  // revision number of this entry, maximum is (256^8)-1
  revision: int;

  /// data stored in this entry, can have a maximum length of 48 bytes
  data: Uint8Array;

  /// signature of this registry entry
  signature: Uint8Array;
}
```

Every registry entry has a 33-byte key.
The first byte indicates which type of public key it is, by default `0xed` for ed25519 public keys.
The other 32 bytes are the ed25519 public key itself.

Every update to a registry entry must contain a **signature** created by the ed25519 keypair referenced in the key.

Nodes only keep the highest **revision** number and reject updates with a lower number.

Because the **data** has a maximum size of 48 bytes, most types of data can't be stored directly in it.
For this reason, registry entries usually contain a CID which then contains the data itself.
The `data` bytes for registry entries which refer to a CID look like this:
```javascript
0x5a 0x261fc4d27f80613c2dfdc4d9d013b43c181576e21cf9c2616295646df00db09fbd95e148
link CID bytes
type
```

## Subscriptions

Nodes can subscribe to specific entries on the peer-to-peer network to get new updates in realtime.