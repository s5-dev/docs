# Metadata formats

This section contains documentation for all metadata formats used and supported by S5.

All formats have a JSON representation for easy creation, debug purposes and editing.

All formats also have a highly optimized serialization representation based on <https://msgpack.org/> used for storing them on S5 including (optional) signatures and timestamp proofs.

JSON Schemas for all formats are available here: <https://github.com/s5-dev/json-schemas>

- [Media Metadata](media.md)
- [Web App Metadata](web-app.md)
- [Directory Metadata](directory.md)