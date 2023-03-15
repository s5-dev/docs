# s5.cx

s5.cx is a web-based tool to securely stream files of any size directly from the S5 network.
File data is NOT proxied by the s5.cx server.

It works by using a service worker that intercepts all raw file requests, fetches the file data from a host on the S5 network and verifies the integrity using BLAKE3/bao in Rust compiled to WASM and running directly inside of the service worker.

The service worker code can be used by any web app to easily stream files from S5 without needing any additional code or libraries in your project. A repository with setup instructions will be published soon.

The service worker is already being used by <https://tube5.app/>.

Here's an example file: <https://s5.cx/uJh9dvBupLgWG3p8CGJ1VR8PLnZvJQedolo8ktb027PrlTT5LvAY.mp4>