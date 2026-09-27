# Go Chat

A simple real-time chat application written in Go. It uses WebSockets and runs in Docker. Open the page in several browser tabs or on several devices, pick a name, and chat.

## Requirements

- Docker with Docker Compose v2

You don't need Go installed locally. The application is compiled inside the Docker build.

## Quick start

```bash
docker compose up --build -d
```

Then open http://localhost:8080.

To see the logs or stop the app:

```bash
docker compose logs -f
docker compose down
```

After changing the code, rebuild with `docker compose up --build -d`.

## Configuration

Settings are passed as environment variables in `docker-compose.yml`.

| Variable          | Default | Description                                                                                      |
|-------------------|---------|--------------------------------------------------------------------------------------------------|
| `PORT`            | `8080`  | Port the server listens on inside the container.                                                 |
| `ALLOWED_ORIGINS` | empty   | Comma-separated extra origins allowed to open WebSockets, e.g. `https://chat.example.com`. Same-origin requests are always allowed. |

If you change `PORT`, update the `ports` mapping in `docker-compose.yml` to match.

## Endpoints

| Path       | Purpose                                   |
|------------|-------------------------------------------|
| `/`        | Chat web page (served from `static/`)     |
| `/ws`      | WebSocket endpoint                        |
| `/healthz` | Health check. Returns `200 ok`.            |

## How it works

The server uses a hub-and-client model:

- **Hub** ([hub.go](hub.go)): a single goroutine that owns the set of connected clients and handles register, unregister and broadcast events on channels. Because only this goroutine touches the client list, no locks are needed. If a client can't keep up with messages, the hub disconnects it.
- **Client** ([client.go](client.go)): each WebSocket connection runs two goroutines. `readPump` validates incoming messages and forwards them to the hub. `writePump` sends outgoing messages and pings the browser to keep the connection alive.
- **Server** ([main.go](main.go)): sets up the HTTP routes and reads the configuration.
- **Front end** ([static/index.html](static/index.html)): a single page written in plain HTML and JavaScript. It reconnects automatically if the connection drops.

Messages are JSON:

```json
{ "username": "alice", "text": "hello", "time": "2026-09-27T07:30:51Z" }
```

The client sends `username` and `text`. The server sets `time`.

## Security measures

- WebSocket connections must come from the same origin or from an origin listed in `ALLOWED_ORIGINS`.
- Input limits: 4 KB per WebSocket frame, 32 characters per username, 1,000 characters per message. Empty or malformed messages are dropped.
- Read and write deadlines, plus ping/pong, close dead connections.
- The front end renders messages with `textContent`, never `innerHTML`, so messages can't inject scripts (XSS).
- The container is built in two stages on a distroless base image (about 14 MB). It runs as a non-root user with a read-only filesystem and `no-new-privileges`.

## Limitations

- Messages are kept only in memory. There's no history, and messages are lost on restart.
- There's no authentication, so anyone can use any display name.
- There's a single chat room.
- The runtime image has no shell, so there's no Docker `HEALTHCHECK`. External monitoring can use `/healthz` instead.
- There's no TLS. For deployment outside localhost, put the app behind a reverse proxy that terminates HTTPS. The front end switches to `wss://` automatically.

## Project structure

```
.
├── main.go             # HTTP server, routes, config
├── hub.go              # Client registry and broadcast loop
├── client.go           # WebSocket connection handling
├── static/index.html   # Web UI
├── Dockerfile          # Multi-stage build
├── docker-compose.yml  # Service definition
├── go.mod / go.sum     # Go module and dependencies
└── .dockerignore
```
