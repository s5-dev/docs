# Streams

> This specification is still a work-in-progress draft. If you spot any issues or have suggestions on how it could be improved, please create an issue here: https://github.com/s5-dev/docs/issues

S5 Streams are pretty much identical to the <registry.md>, with the main difference being that S5 Nodes are expected to store old stream messages, not just the latest one (like the registry). The serialized data structure of stream messages is identical to registry entries, only the constant prefix byte is changed to `0x08` instead of `0x07` (for the registry) and it uses big-endian encoding for the `u64` revision number.

In addition to storing previous stream messages, it's also possible to query them using range requests. For example, "Please send me all stream messages with this seq/ts or greater, including any messages you'll receive in the future".

The revision number of a stream message is a `u64` and is usually composed of a `u32` unix timestamp and a `u32` sequence number. This is however application-specific, so you can also use a slightly bigger timestamp and some random bytes to build the revision number if you prefer. S5 Nodes only care about the complete `u64` revision number for ordering and range requests.

When serialized, stream messages can also contain the blob bytes referenced by the hash in the stream message to optimize lookup latency for small data packets.
