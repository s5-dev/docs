# Identity System

> This specification is still a work-in-progress draft. If you spot any issues or have suggestions on how it could be improved, please create an issue here: https://github.com/s5-dev/docs/issues

There are two types of identity, private and public ones. Currently, S5 only supports private identites. This means that at the moment, the S5 root identity (stored in the seed phrase) is only used to derive private secrets, like the ones managing access and encryption for the private file system. While you can share directories from your private file system with others, or even create a public file system, S5 does not manage one public user identity key which then links to the entrypoints to these or stores other public data.

Instead, for all use cases which require public identites (like for example publishing your favorite files in a public file system under your name), S5 will use the AT Protocol (built by Bluesky). The AT Protocol uses DIDs for public identity identifiers and S5 will likely store private key material used for your public AT Protocol identites in the future, to make it very easy to sign in to both (S5 and ATProto) at once using only your seed phrase. Your public AT Protocol identity (which might also be used as your social media presence) will then support S5-specific features like posting files or directories hosted on S5.

Regarding authentication, it's also important to note that almost all operations which only involve reading data (like listing directories or downloading a file) don't require an identity on S5.

## Seed Phrases

S5 uses a custom algorithm for seed phrases:

The wordlist consists of 1024 unique words: [https://github.com/s5-dev/lib5/blob/main/lib/src/seed/wordlist.dart](https://github.com/s5-dev/lib5/blob/main/lib/src/seed/wordlist.dart)

Every word on the list has a unique 3-letter prefix - this makes it possible to change a word in your seed phrase to whatever you want, as long as the first 3 characters stay the same.

So every word contains 10 bits of entropy, which means that we need 13 words for our 128 bits of entropy. Finally, there are two extra words containing a checksum (blake3), which should make it easier to recover your identity if you lost some words of your seed phrase. As a result, S5 seed phrases are always 15 words long.

The current implementation of seed generation and validation is available at <https://github.com/s5-dev/lib5/blob/main/lib/src/seed/seed.dart>, the algorithm will be added to this spec page in the future (still subject to small changes).

## Derivation

After the root identity secret is available (from the seed phrase), it's used to derive a number of keys. S5 applications are expected to only (securely) store the derived keys they actually need for their features (like the file system or accounts) and drop all other keys, including the root identity secret.

The derivation algorithm is described in <key-derivation.md> and the derivation paths for different use cases is implemented in <https://github.com/s5-dev/lib5/blob/main/lib/src/identity/identity.dart>. The paths will be added to this spec after the final decision on which ones are actually needed (and which ones can be removed due to no active use) is made.
