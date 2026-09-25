# linode-docker

linode-docker is a small Docker Compose setup for running a self-hosted live video streaming server, for example on a Linode VPS. A broadcaster sends an RTMP stream from software such as OBS to an nginx server with the RTMP module. nginx asks a separate Node.js service whether the stream key is allowed before accepting the stream, then repackages the video as HLS segments and serves them over HTTP together with a one-page web player that uses hls.js. It suits a single private stream for a small audience without relying on a streaming platform; the player page is titled for a World Cup stream. The stack is two containers: `tiangolo/nginx-rtmp` with a custom `nginx.conf`, and an Express app on Node 12 for the `on_publish` check. It is a working prototype from November 2022 with a single hardcoded stream key and stream name, and it has not been updated since.

> Prototype, built in November 2022. Not actively maintained.

## Features

- RTMP ingest on port 1935 at the `live` application
- Stream key check through nginx `on_publish`, which posts to the `auth` service (`POST /auth`, form field `key`); any key other than the one in `auth/server.js` gets `403` and the stream is rejected
- HLS output with 10-second fragments and a 5-minute playlist, written to `./data` on the host
- HTTP server on container port 8080 (published as port 80) serving the player at `/` and the HLS files at `/hls` with `Cache-Control: no-cache` and CORS enabled
- Player page (`rtmp/index.html`) that plays `/hls/test.m3u8` with hls.js, or natively where the browser supports HLS

## Tech stack

Docker Compose · nginx-rtmp (`tiangolo/nginx-rtmp`) · HLS · Node.js 12 · Express · nodemon · hls.js

## Getting started

Prerequisites: Docker with Docker Compose.

```bash
docker-compose build
docker-compose up
```

Then:

1. Set the accepted stream key in `auth/server.js` before exposing the server; the committed value is a demo placeholder.
2. In OBS, open **Settings → Stream**, choose a custom service and set:
   - Server: `rtmp://localhost:1935/live` (use the server's address when running remotely)
   - Stream key: `test?key=<your key>`. The stream name must be `test`, because the player loads `/hls/test.m3u8`.
3. Start streaming, then open `http://localhost/` (port 80) to watch.

Ports used on the host: `1935` (RTMP) and `80` (HTTP). The auth service listens on port 8000 inside the Compose network only.

## Project structure

```text
docker-compose.yml   rtmp and auth services; ./data mounted as /tmp/hls
rtmp/
├── Dockerfile       tiangolo/nginx-rtmp with the config and player copied in
├── nginx.conf       RTMP application "live", HLS settings, HTTP server on 8080
└── index.html       hls.js player for /hls/test.m3u8
auth/
├── Dockerfile       node:12 image running "npm start" (nodemon server.js)
├── package.json
└── server.js        POST /auth: 200 for the accepted key, 403 otherwise
```

## How it works

1. OBS connects to `rtmp://<host>:1935/live/test?key=...`.
2. nginx-rtmp calls `http://auth_server:8000/auth` with the publish arguments; the Express app compares `key` with its accepted value and answers `200` or `403`.
3. On `200`, nginx writes HLS segments and `test.m3u8` to `/tmp/hls`, which is mounted from `./data`.
4. Viewers load `index.html`, which fetches `/hls/test.m3u8` from the same nginx server.

## Limitations

- One accepted stream key, hardcoded in `auth/server.js`; there is no user database.
- The player is fixed to the stream name `test`.
- No HTTPS; put a reverse proxy with TLS in front for public use.
- The auth container runs under nodemon on Node 12, which is end-of-life.
