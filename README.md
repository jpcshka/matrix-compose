# Private Matrix Server based on Synapse

A ready-to-use configuration for deploying a private Matrix homeserver via Docker Compose (without federation).

> Also available in: [Русский](README.ru.md)

## Table of Contents

1. [Stack](#stack)
2. [Repository Structure](#repository-structure)
3. [Requirements](#requirements)
4. [DNS Setup](#dns-setup)
5. [Quick Start](#quick-start)
6. [Components](#components)
    - [Synapse](#synapse)
    - [Caddy](#caddy)
    - [LiveKit](#livekit)
    - [lk-jwt-service](#lk-jwt-service)


---

## Stack

- **[Matrix Synapse](https://github.com/element-hq/synapse)**: The main homeserver, developed by The Matrix.org Foundation in Python.
- **[Caddy](https://github.com/caddyserver/caddy)**: Reverse proxy server, issues and manages Let's Encrypt certificates. Developed by Matt Holt in Go.
- **[LiveKit](https://github.com/livekit/livekit)**: Media server for MatrixRTC, written in Go by LiveKit Inc.
- **[lk-jwt-service](https://github.com/element-hq/lk-jwt-service)**: Authorization service for MatrixRTC, developed by The Matrix.org Foundation in Go.


---


## Repository Structure

```text
.
├── caddy/              # Caddy configuration (Caddyfile)
├── livekit/            # LiveKit configuration example (livekit.yaml.example)
├── synapse/            # Synapse with S3 support (Dockerfile)
├── static/             # Static files for the placeholder page (html, css, js)
├── .env.example        # Environment variables example
└── compose.yaml        # Main compose file
```


---


## Requirements

- Linux server (Debian 13 recommended)
- A domain name
- Open ports:
  - `80/tcp`
  - `443/tcp`
  - `443/udp`
  - `7881/tcp` (optional)
  - `50100-50101/udp` (optional)

> The list of required ports depends on your MatrixRTC (LiveKit) configuration.

### Recommended Server Configuration

| Resource | Recommendation |
|---|---|
| CPU | 2-4 vCPU |
| RAM | 2-8 GB |
| Disk | 20+ GB |
| Network | Public IPv4 |

> At least 4 GB RAM is recommended for MatrixRTC.


---

## DNS Setup

Create A records for your Matrix homeserver and MatrixRTC domains pointing to your server's IP address.

```text
example.com          A    203.0.113.1
matrix.example.com   A    203.0.113.1
livekit.example.com  A    203.0.113.1
```

> All services run on the same server, so all DNS records should point to the same IP address.

---


## Quick Start

1. Copy and fill in the environment variables:

```bash
cp .env.example .env
```

2. Read the [Components](#components) section and configure each service according to the documentation.

3. Start:

```bash
docker compose up -d
```

> Make sure your DNS records are configured and ports are open before starting — Caddy won't issue certificates without external access.

---


## Components

### Synapse

Matrix homeserver based on the official [matrixdotorg/synapse](https://hub.docker.com/r/matrixdotorg/synapse) image with the [synapse-s3-storage-provider](https://github.com/matrix-org/synapse-s3-storage-provider) module added for storing media in S3-compatible storage.

Configuration files, keys, and media are stored in `/var/lib/matrix-compose/data`.

Create the directory:
```bash
mkdir -p /var/lib/matrix-compose/data
```

Generate configuration files:

```bash
docker run -it --rm \
    --mount type=bind,src=/var/lib/matrix-compose/data,dst=/data \
    -e SYNAPSE_SERVER_NAME=example.com \
    -e SYNAPSE_REPORT_STATS=no \
    matrixdotorg/synapse:latest generate

```

Add the following line to `homeserver.yaml`:

```yaml
public_baseurl: "https://matrix.example.com/"
```

You also need to configure the Postgres connection:

```yaml
database:
  name: psycopg2
  args:
    user: synapse_user
    password: your_strong_password
    dbname: synapse
    host: 1.2.3.4
    port: 5432
    cp_min: 5
    cp_max: 10
    keepalives_idle: 10
    keepalives_interval: 10
    keepalives_count: 3
```

To enable MatrixRTC calls, you need to enable federation or openid:

```yaml
listeners:
  - port: 8008
    resources:
    - compress: false
      names:
      - client
#      - federation
      - openid # <---
    tls: false
    type: http
    x_forwarded: true
```

You also need to enable MSCs (Matrix spec proposals) and specify the LiveKit server domain:

```yaml
experimental_features:
  msc4222_enabled: true
  msc4354_enabled: true

max_event_delay_duration: 24h
rc_message:
  per_second: 0.5
  burst_count: 30
rc_delayed_event_mgmt:
  per_second: 1
  burst_count: 20

matrix_rtc:
  transports:
  - type: livekit
    livekit_service_url: https://livekit.example.com/livekit/jwt
```

To enable S3 storage, configure the storage provider:

```yaml
media_storage_providers:
- module: s3_storage_provider.S3StorageProviderBackend
  store_local: True
  store_remote: True
  store_synchronous: True
  config:
    bucket: <S3_BUCKET_NAME>
    region_name: <S3_REGION_NAME>
    endpoint_url: <S3_LIKE_SERVICE_ENDPOINT_URL>
    access_key_id: <S3_ACCESS_KEY_ID>
    secret_access_key: <S3_SECRET_ACCESS_KEY>
    session_token: <S3_SESSION_TOKEN>
```

**See also:**
- [Synapse Documentation](https://element-hq.github.io/synapse/latest/)
- [Synapse image on Dockerhub](https://hub.docker.com/r/matrixdotorg/synapse)
- [synapse-s3-storage-provider](https://github.com/matrix-org/synapse-s3-storage-provider)
- [Self-Hosting MatrixRTC](https://github.com/element-hq/element-call/blob/livekit/docs/self_hosting.md)


---


### Caddy

Reverse proxy based on the official [caddy](https://hub.docker.com/_/caddy) image. Issues and manages TLS certificates for each subdomain, serves `.well-known` endpoints for Matrix, redirects all other requests to example.com, and serves static files.

By default, Caddy does not set a limit on request body size. An explicit limit of 300MB is configured:

```caddy
request_body {
  max_size 300MB
}
```

The limit must match the limit in `homeserver.yaml`:

```yaml
max_upload_size: 300M
```

For QUIC (HTTP/3) and LiveKit WebRTC traffic to work correctly, you need to increase the UDP buffers at the kernel level.

Apply immediately (until reboot):

```bash
sudo sysctl -w net.core.rmem_max=7500000
sudo sysctl -w net.core.wmem_max=7500000
```

To persist after reboot, add to `/etc/sysctl.conf`:

```ini
net.core.rmem_max=7500000
net.core.wmem_max=7500000
```

The value 7500000 is recommended by the [quic-go](https://github.com/quic-go/quic-go/wiki/UDP-Buffer-Sizes) library used internally by Caddy.

**See also:**
- [Caddy Documentation](https://caddyserver.com/docs/)
- [Hardened image](https://hub.docker.com/hardened-images/catalog/dhi/caddy)


---


### LiveKit

Media server for MatrixRTC. LiveKit does not support environment variables for most configuration parameters, so all settings are defined directly in `livekit/livekit.yaml`. Copy the example and fill it in:

```bash
cp livekit/livekit.yaml.example livekit/livekit.yaml
```

Instead of opening dozens of ports, UDP mux is used. The number of ports must be **no less than the number of CPU cores on the server**:

```yaml
rtc:
  udp_port: 50100-50101
```

You also need to expose these ports in `compose.yaml`:

```yaml
livekit:
  ports:
    - "50100-50101:50100-50101/udp"
```

If you use multiple LiveKit servers, Redis is required.

**See also:**
- [LiveKit Documentation](https://docs.livekit.io/transport/self-hosting/deployment/)
- [Configuration example](https://github.com/livekit/livekit/blob/master/config-sample.yaml)


---

### lk-jwt-service

Authorization service for MatrixRTC based on the [official image](https://github.com/element-hq/lk-jwt-service/pkgs/container/lk-jwt-service). Acts as middleware between the client and LiveKit: issues JWT tokens that authorize participation in a call. All parameters are passed via environment variables in `.env` — no separate configuration file is required.
