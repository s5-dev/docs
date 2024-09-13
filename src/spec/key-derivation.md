# Key Derivation

S5 uses a very simple and fast (but secure) key derivation function for identity, accounts and the file system (only scoped to a single directory).

The `base` key is always random; in the file system it's randomly generated for every newly created directory, for accounts it's derived in multiple steps from the identity root secret (which is randomly generated).

```rs
fn derive_hash(base: &[u8; 32], tweak: &[u8; 32]) -> [u8; 32] {
    let mut hasher = blake3::Hasher::new();
    hasher.update(base);
    hasher.update(tweak);
    *hasher.finalize().as_bytes()
}

fn derive_hash_int(base: &[u8; 32], tweak: u16) -> [u8; 32] {
    let mut tweak_bytes = [0u8; 32];
    tweak_bytes[..2].copy_from_slice(&tweak.to_le_bytes());
    derive_hash(base, &tweak_bytes)
}
```