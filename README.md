
# Immich Machine Learning with Tailscale Sidecar — Docker Setup

---

## Overview

This repository contains a Docker Compose setup to run **Immich Machine Learning (ML)** service alongside a **Tailscale** sidecar container for secure mesh networking. This allows your ML service to be securely accessed over your private Tailnet without exposing public ports or proxies.

---

## Table of Contents

- [Immich Machine Learning with Tailscale Sidecar — Docker Setup](#immich-machine-learning-with-tailscale-sidecar--docker-setup)
  - [Overview](#overview)
  - [Table of Contents](#table-of-contents)
  - [Project Summary](#project-summary)
  - [Prerequisites](#prerequisites)
  - [Repository Structure](#repository-structure)
  - [Getting Started](#getting-started)
    - [1. Clone the repo](#1-clone-the-repo)
    - [2. Prepare `.env` file](#2-prepare-env-file)
    - [3. Create necessary folders](#3-create-necessary-folders)
    - [4. Deploy with Docker Compose](#4-deploy-with-docker-compose)
  - [Configuration Details](#configuration-details)
    - [`.env` file explained](#env-file-explained)
    - [Docker ports](#docker-ports)
    - [Tailscale sidecar container](#tailscale-sidecar-container)
  - [How it works](#how-it-works)
  - [Troubleshooting](#troubleshooting)
    - [Tailscale container keeps restarting](#tailscale-container-keeps-restarting)
    - [Cannot access Immich ML over Tailscale IP](#cannot-access-immich-ml-over-tailscale-ip)
    - [`.env` variables not picked up](#env-variables-not-picked-up)
  - [Frequently Asked Questions (FAQs)](#frequently-asked-questions-faqs)
  - [Security considerations](#security-considerations)
  - [Useful commands](#useful-commands)
  - [Final notes](#final-notes)

---

## Project Summary

- Runs Immich ML in a container for image recognition tasks.  
- Runs Tailscale as a **sidecar container** within the same Docker network/host, exposing a private Tailnet IP.  
- Avoids exposing open public ports — access Immich ML securely over your Tailnet.  
- Uses volumes for model cache persistence.  
- Restart policies for resilience.  

---

## Prerequisites

- Linux host (recommended) or compatible Docker environment with `/dev/net/tun` support.  
  > **Note:** macOS with Docker Desktop generally does not support `/dev/net/tun` inside containers, so Tailscale sidecar may fail to start.  
- Docker Engine (20.10+) and Docker Compose installed.  
- Tailnet and Tailscale account — to generate auth keys and manage devices.  
- Basic Docker and CLI familiarity.  

---

## Repository Structure

```

immich-ml/
├── docker-compose.yml       # Main Docker Compose stack
├── tailscale.json           # Tailscale config file (JSON)
├── .env                    # Environment variables file (contains secrets)
├── model-cache/            # Persistent cache volume for ML models
├── state/                  # State folder for tailscale container
└── README.md               # This document

````

---

## Getting Started

### 1. Clone the repo

```bash
git clone https://github.com/yourusername/immich-ml.git
cd immich-ml
````

### 2. Prepare `.env` file

Create a `.env` file in the root directory with the following variables:

```bash
# Your project service name (used for container naming)
service_name=immich_ml

# Immich Machine Learning image version tag (e.g. release, or specific tag)
IMMICH_VERSION=release

# Tailscale auth key from your Tailnet admin console
TS_AUTHKEY=tskey-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Upload location (optional for ML, usually your data path)
UPLOAD_LOCATION=/path/to/your/upload/location
```

> **IMPORTANT:** Keep `.env` private — do NOT commit this file to public repos!

### 3. Create necessary folders

Make sure these folders exist with proper permissions:

```bash
mkdir -p model-cache
mkdir -p ${service_name}_tailscale/state
```

### 4. Deploy with Docker Compose

```bash
docker-compose up -d
```

Check that containers are running:

```bash
docker ps
```

You should see `immich_ml_learning` and `immich_ml_tailscale` containers up and running.

---

## Configuration Details

### `.env` file explained

| Variable          | Description                                     | Example                |
| ----------------- | ----------------------------------------------- | ---------------------- |
| `service_name`    | Prefix for container names                      | immich\_ml             |
| `IMMICH_VERSION`  | Tag for the Immich ML Docker image              | release                |
| `TS_AUTHKEY`      | Your Tailscale authentication key               | tskey-abcdef1234567890 |
| `UPLOAD_LOCATION` | Path where images or data are stored (optional) | /data/uploads          |

### Docker ports

* Immich ML service **does not expose ports directly** in this setup; it runs inside Docker network accessible via Tailnet IP.
* Tailscale container listens internally for VPN and sidecar functions, no public ports exposed.

### Tailscale sidecar container

* Runs `tailscaled` daemon managing VPN connection.
* Uses `/dev/net/tun` device — ensure your host supports this.
* Mounts persistent `state` volume for tailscale state files.
* Uses environment variables for auth key, hostname, extra args, and config JSON.
* Advertises a container tag for network filtering or ACLs in Tailnet admin.

---

## How it works

* The **tailscale container** establishes a secure encrypted connection to your Tailnet, creating a private IP accessible only to your trusted devices.
* The **immich-machine-learning container** runs your ML service, accessible via the tailscale private IP and internal Docker network.
* No ports are exposed publicly, enhancing security by isolating access to your Tailnet devices only.
* Persistent volumes store ML model cache and tailscale state for durability.

---

## Troubleshooting

### Tailscale container keeps restarting

* Check logs:

  ```bash
  docker logs ${service_name}_tailscale
  ```
* Common issues:

  * Invalid or missing `TS_AUTHKEY` in `.env`.
  * `/dev/net/tun` device missing or inaccessible (especially on macOS).
  * Incorrect or missing `tailscale.json` config file.

### Cannot access Immich ML over Tailscale IP

* Verify tailscale container status and IP with:

  ```bash
  docker exec -it ${service_name}_tailscale tailscale ip
  ```
* Confirm the ML container is running and listening internally.
* Confirm firewall rules on your host are not blocking connections.
* Ensure your client device is connected to the same Tailnet and can ping the Tailscale IP.

### `.env` variables not picked up

* Make sure `.env` is in the same directory as `docker-compose.yml`.
* Restart the containers after changing `.env` to apply changes:

  ```bash
  docker-compose down && docker-compose up -d
  ```

---

## Frequently Asked Questions (FAQs)

**Q: Can I run Tailscale sidecar on macOS?**
A: Usually no, because Docker Desktop on macOS does not expose `/dev/net/tun` device. You need a Linux host or a VM.

**Q: How do I generate a Tailscale auth key?**
A: Go to your [Tailscale Admin Console](https://login.tailscale.com/admin/settings/keys), create an ephemeral or reusable auth key.

**Q: Why no ports are exposed on ML container?**
A: Access is secured via Tailnet private IPs, no public port exposure required.

**Q: How do I update Immich ML image?**
A: Change `IMMICH_VERSION` in `.env` then:

```bash
docker-compose pull
docker-compose up -d
```

---

## Security considerations

* Keep your `.env` file and `TS_AUTHKEY` secret.
* Do not expose ports publicly unless necessary.
* Use Tailnet ACLs to restrict access by tags or device IDs.
* Run containers with minimal privileges (`no-new-privileges` enabled).

---

## Useful commands

```bash
# View tailscale container logs
docker logs ${service_name}_tailscale

# View ML container logs
docker logs ${service_name}_learning

# Restart containers
docker-compose restart

# Stop and remove containers
docker-compose down

# Check Tailscale status inside container
docker exec -it ${service_name}_tailscale tailscale status

# Get Tailscale IP of container
docker exec -it ${service_name}_tailscale tailscale ip
```

---

## Final notes

This setup allows you to run Immich ML securely over your Tailnet using container-level Tailscale sidecar VPN. It avoids port exposure and ensures your ML service is accessible only on your private mesh network.

Keep this README handy for troubleshooting, reference, and easy recreation of the environment anytime!

---

If you have any questions or need help, feel free to open an issue or contact me.

Happy secure photo machine learning! 🚀
