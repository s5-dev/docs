# Peer-to-peer

S5 uses a peer-to-peer network to find providers who serve a specific file. Compared to IPFS, S5 does NOT transfer or exchange file data between
peers directly. Instead, the p2p network is only meant to find storage locations and download links for a specific file hash. This has some advantages:

- Because only lightweight queries for hashes (34 bytes) and responses (only a short download link, usually less than 256 bytes) are sent over the p2p network, it's extremely lightweight and very scalable.
- Existing highly optimized HTTP-based software and infrastructure can be used for the file delivery itself, reducing costs significantly and making download more efficient. Also keeps peers lightweight.
- Because S5 uses the HTTP/HTTPS protocol (support for more is planned), existing download links or files mirrors can be directly provided on S5 without needing to re-upload them - even if the one who provides it on the network is not the same one hosting it.

## Peer discovery

Right now S5 uses a configurable list of initial peers with their connection strings (protocol, ip address, port) to connect to the network.
After connecting to a new peer, peers send a list of all other peers they know about to the new peer.

## Supported P2P Protocols

- WebSocket (`wss://`)

## Planned P2P Protocols

- iroh QUIC Connections (<https://iroh.computer/docs/layers/connections>)

## Node/peer IDs

Every node has a unique (random) **ed25519 keypair**. This keypair is used to sign specific responses like **provide operations**, which contain a specific storage location and download link for a queried hash.
Because the message itself contains the signature, all peers can also relay queries and responses without being trusted to not tamper with them.

## Node scores

Every node keeps a local score for every other node/peer it knows of. This score is calculated based on the number of valid and useful responses by a node compared to the number of bad or invalid responses.
The score also depends on the total number of responses, so a node with 1000 correct and 50 wrong responses has a better score than a node with 5 correct out of only 5 total responses for example.

The algorithm can be found here: [lib5:score.dart](https://github.com/s5-dev/lib5/blob/main/lib/src/util/score.dart)

Node scores are used to decide which download links to try first if multiple are available for the same file hash.