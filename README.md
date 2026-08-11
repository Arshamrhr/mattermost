# Mattermost — Local Communication Infrastructure for Internet Blackouts

A self-hosted **Mattermost** deployment designed to provide a reliable internal communication platform during severe Internet restrictions and Internet blackouts in Iran.

This project was deployed as an emergency communication solution when access to many external communication services and infrastructure was unavailable or severely restricted. By hosting the entire communication stack on infrastructure located inside Iran, the team was able to maintain communication using the national network even when normal Internet connectivity was unavailable.

---

## Why This Project Exists

During an Internet blackout, access to many common communication platforms and external services can become unavailable.

We needed a communication platform that:

* Could be **self-hosted**
* Could run entirely on infrastructure inside Iran
* Could remain accessible through the **Iranian national network** when international Internet connectivity was unavailable
* Did not depend on external SaaS communication platforms
* Could support messaging, file sharing, and voice/video communication
* Could be deployed without relying on direct access to Docker Hub

For this reason, we deployed **Mattermost** on an internally accessible server.

The goal was simple:

> **Keep the team connected when normal Internet communication was no longer available.**

---

## Architecture

The deployment consists of the following components:

```text
                    ┌─────────────────────────┐
                    │       Users / Team      │
                    │   Web / Desktop / App   │
                    └────────────┬────────────┘
                                 │
                                 │
                         Iranian Network
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │       Mattermost        │
                    │    Communication UI     │
                    └────────────┬────────────┘
                                 │
                ┌────────────────┼────────────────┐
                │                │                │
                ▼                ▼                ▼
          ┌───────────┐   ┌────────────┐   ┌───────────┐
          │ PostgreSQL│   │   MinIO    │   │  Coturn   │
          │  Database │   │ File Store │   │ TURN/STUN │
          └───────────┘   └────────────┘   └─────┬─────┘
                                                  │
                                                  │
                                         Voice / Call Traffic
```

### Components

#### Mattermost

Mattermost is the main communication platform.

It provides:

* Team messaging
* Channels
* Direct messages
* File sharing
* Voice/video communication
* Self-hosted administration

The official Mattermost project describes the platform as a self-hosted collaboration platform supporting communication, voice calling, screen sharing, and other collaboration capabilities.

#### PostgreSQL

PostgreSQL is used as the primary database for Mattermost.

The Docker Compose deployment runs PostgreSQL as a separate container and connects Mattermost to it through the Docker network.

#### MinIO

MinIO is used as an S3-compatible object storage backend for Mattermost file uploads.

This allows uploaded files and attachments to be stored separately from the Mattermost application container.

#### Coturn

Coturn is used as the **STUN/TURN infrastructure for real-time communication traffic**.

The important reason for running Coturn in this architecture is to keep call/media traffic within infrastructure located inside Iran.

Instead of relying on an external TURN service, the Mattermost deployment uses a locally hosted Coturn server so that real-time communication traffic can be relayed through our own infrastructure.

Coturn is an open-source implementation of the STUN/TURN protocols and is specifically designed for NAT traversal and relaying real-time media traffic.

In this deployment, Coturn uses host networking:

```yaml
coturn:
  image: coturn/coturn:latest
  network_mode: "host"
```

This allows Coturn to directly use the server's network stack, which is particularly useful for TURN traffic and its associated UDP port ranges.

---

# Docker Registry Considerations

One of the major challenges during the deployment was that **Docker Hub was not reliably accessible from the Iranian network**.

Normally, running:

```bash
docker compose up -d
```

would cause Docker to pull images directly from their configured registries.

During the Internet blackout/restricted connectivity scenario, this was not a reliable option.

Therefore, Docker images had to be obtained through **Iranian Docker registries or other accessible sources**.

There are two supported approaches.

---

## Method 1 — Change the Image Registry in `docker-compose.yml`

The first approach is to modify the `image:` fields in `docker-compose.yml` and replace the original registry/image path with an image hosted on an accessible Iranian registry.

For example, if an image is available through an Iranian registry:

```yaml
services:
  mattermost:
    image: <iranian-registry>/mattermost/mattermost-team-edition:latest

  postgres:
    image: <iranian-registry>/postgres:18-alpine

  minio:
    image: <iranian-registry>/minio/minio:latest

  coturn:
    image: <iranian-registry>/coturn/coturn:latest
```

For example, using an internal mirror or a registry such as **Arvan**:

```yaml
services:
  mattermost:
    image: <arvan-registry>/mattermost/mattermost-team-edition:latest
```

The exact registry URL and image path depend on where the images are currently hosted.

After changing the image references:

```bash
docker compose up -d
```

Docker will pull the images from the configured registry instead of attempting to access Docker Hub.

---

# Method 2 — Pull and Retag the Images

The second approach is useful when the images can be downloaded from an accessible registry, but the `docker-compose.yml` is expected to keep the original image names.

Docker image names can be **retagged locally**.

For example, suppose the Compose file expects:

```yaml
image: mattermost/mattermost-team-edition:latest
```

but the image is available from an Iranian registry:

```bash
docker pull <iranian-registry>/mattermost/mattermost-team-edition:latest
```

After pulling the image, tag it with the exact name expected by Compose:

```bash
docker tag \
  <iranian-registry>/mattermost/mattermost-team-edition:latest \
  mattermost/mattermost-team-edition:latest
```

The same process can be applied to the other images.

For example:

```bash
docker pull <iranian-registry>/postgres:18-alpine

docker tag \
  <iranian-registry>/postgres:18-alpine \
  postgres:18-alpine
```

And:

```bash
docker pull <iranian-registry>/minio/minio:latest

docker tag \
  <iranian-registry>/minio/minio:latest \
  minio/minio:latest
```

For Coturn:

```bash
docker pull <iranian-registry>/coturn/coturn:latest

docker tag \
  <iranian-registry>/coturn/coturn:latest \
  coturn/coturn:latest
```

Once all required images exist locally with the names expected by `docker-compose.yml`, simply run:

```bash
docker compose up -d
```

Docker Compose will find the images locally and does not need to download them again.

---

# Deployment

Clone the repository:

```bash
git clone https://github.com/Arshamrhr/mattermost.git
cd mattermost
```

Create the external Docker network required by the Compose configuration:

```bash
docker network create my-shared-network
```

Then make sure all required images are available locally or that the image references point to an accessible registry.

Finally:

```bash
docker compose up -d
```

Check the running containers:

```bash
docker compose ps
```

And inspect logs if necessary:

```bash
docker compose logs -f
```

For a specific service:

```bash
docker compose logs -f mattermost
```

---

# Offline / Restricted-Network Deployment Strategy

The key idea behind this deployment is that **the application infrastructure itself does not need to be dependent on the international Internet once the required images and configuration are available locally.**

A typical deployment workflow is:

```text
                 Before Blackout
                       │
                       ▼
          Obtain required Docker images
                       │
                       ▼
           Store images in accessible
              Iranian registry
                       │
                       ▼
              Deploy infrastructure
                       │
                       ▼
          ┌────────────────────────┐
          │  Server inside Iran    │
          │                        │
          │  Mattermost            │
          │  PostgreSQL            │
          │  MinIO                 │
          │  Coturn                │
          └───────────┬────────────┘
                      │
                      ▼
              Iranian Network
                      │
                      ▼
                  Team Users
```

This approach removes a major dependency on external communication platforms and external container registries.

---

# Important Notes

### Docker Images

The default Compose configuration references images such as:

```text
mattermost/mattermost-team-edition
postgres
minio/minio
coturn/coturn
```

If Docker Hub is inaccessible, use either:

1. An accessible Iranian registry directly in `docker-compose.yml`, or
2. Pull the images from another accessible registry and retag them locally.

### Coturn

Coturn is intentionally deployed separately from the normal Docker bridge network and uses:

```yaml
network_mode: "host"
```

This is important for handling real-time media traffic and TURN networking.

### Data Persistence

The deployment uses Docker volumes for persistent application data:

```text
postgres_data
minio_data
mattermost_config
mattermost_data
mattermost_logs
mattermost_plugins
```

Do not remove these volumes unless you intentionally want to delete the corresponding persistent data.

---

# Project Motivation

This project is more than just a Mattermost Docker deployment.

It was created as a practical response to a real operational problem:

**What happens when the Internet goes down and the team still needs to communicate?**

By hosting Mattermost and its supporting services on infrastructure reachable through the Iranian network, we were able to maintain an internal communication channel during an Internet blackout when many normal communication tools were unavailable.

The project demonstrates a simple principle:

> **Critical communication infrastructure should not have a single dependency on external Internet services.**

---

## References

* [Mattermost](https://github.com/mattermost/mattermost?utm_source=chatgpt.com)
* [Mattermost Docker Deployment](https://github.com/mattermost/docker?utm_source=chatgpt.com)
* [Coturn](https://github.com/coturn/coturn?utm_source=chatgpt.com)
* [This Deployment Repository](https://github.com/Arshamrhr/mattermost?utm_source=chatgpt.com)
