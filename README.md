
# Endpoint Management

Fleet, Salt, Santa, LuLu and endpoint configuration management.

## Fleet Server — Local Deployment on Apple Silicon

This guide describes how to build and run **Fleet Server 4.95.0** locally on a Mac with Apple Silicon (ARM64), using OrbStack or Docker Desktop.

The deployment is intended for local development and testing, with future access from devices on a private local network. It is not configured for public internet exposure.

The current configuration uses:

- Fleet Server 4.95.0, built natively for ARM64
- MySQL 8
- Redis 6
- Docker Compose
- A temporary self-signed TLS certificate

### 1. Requirements

- macOS on Apple Silicon (ARM64)
- OrbStack or Docker Desktop
- Docker Compose with Buildx
- Git
- OpenSSL

All commands below are executed from the repository root unless otherwise specified.

### 2. Build Fleet for ARM64

Fleet 4.95.0 is built from source using the Dockerfile provided in this repository.

Clone the Fleet source code:

```bash
mkdir -p src

git clone \
  --branch fleet-v4.95.0 \
  --depth 1 \
  https://github.com/fleetdm/fleet.git \
  src/fleet-arm64
```

Build the ARM64 Docker image:

```bash
docker buildx build \
  --platform linux/arm64 \
  --file compose/Dockerfile.fleet-arm64 \
  --tag fleet-local:4.95.0-arm64 \
  --load \
  src/fleet-arm64
```

The resulting image is:

```text
fleet-local:4.95.0-arm64
```

Verify the image:

```bash
docker image inspect \
  fleet-local:4.95.0-arm64 \
  --format '{{.Os}}/{{.Architecture}}'
```

Expected output:

```text
linux/arm64
```

The `src/` directory contains third-party source code used for local builds and is excluded from Git.

### 3. Configure Environment Variables

Navigate to the Compose directory and create the local environment file:

```bash
cd compose
cp .env.example .env
```

Edit `compose/.env` and replace the placeholder MySQL passwords:

```dotenv
MYSQL_ROOT_PASSWORD=replace_with_secure_password
MYSQL_PASSWORD=replace_with_secure_password
```

Keep the existing database and username settings from `.env.example` unless customization is required.

#### Generate the Fleet server secret

Fleet requires an internal server secret of at least 32 bytes.

Generate a cryptographically secure value:

```bash
openssl rand -hex 32
```

Replace the placeholder in `compose/.env`:

```dotenv
FLEET_SERVER_PRIVATE_KEY=change_this_private_key
```

with the generated value.

For example, the final entry should have this format:

```dotenv
FLEET_SERVER_PRIVATE_KEY=<generated_64_character_hex_value>
```

This secret is used internally by Fleet. It is **not** the TLS private key.

Do not commit `.env` or expose the generated secret.

#### Configure local HTTPS

Ensure the following settings are present in `compose/.env`:

```dotenv
FLEET_SERVER_ADDRESS=0.0.0.0
FLEET_SERVER_PORT=1337
FLEET_SERVER_TLS=true
FLEET_SERVER_CERT=/fleet/fleet.crt
FLEET_SERVER_KEY=/fleet/fleet.key
```

The `.env` file is excluded from Git.

### 4. Generate a Self-Signed TLS Certificate

For initial testing on the Mac running Fleet, generate a self-signed certificate for:

- `localhost`
- `127.0.0.1`

From the `compose/` directory:

```bash
mkdir -p certs

openssl req \
  -x509 \
  -newkey rsa:3072 \
  -sha256 \
  -nodes \
  -days 365 \
  -keyout certs/fleet.key \
  -out certs/fleet.crt \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

chmod 600 certs/fleet.key
chmod 644 certs/fleet.crt
```

This creates:

```text
compose/
└── certs/
    ├── fleet.crt
    └── fleet.key
```

- `fleet.crt` — self-signed server certificate
- `fleet.key` — TLS private key

The `compose/certs/` directory is excluded from Git.

**Important: Generate both files before starting Docker Compose.**

#### Certificate mounts

The Fleet service in `compose/compose.yaml` mounts the certificate and private key as individual read-only files:

```yaml
volumes:
  - data:/fleet
  - logs:/logs
  - vulndb:${FLEET_VULNERABILITIES_DATABASES_PATH}
  - ./certs/fleet.crt:/fleet/fleet.crt:ro
  - ./certs/fleet.key:/fleet/fleet.key:ro
```

The `data:/fleet` named volume stores Fleet data.

The certificate bind mounts make the TLS files available inside the container at:

```text
/fleet/fleet.crt
/fleet/fleet.key
```

These paths must match the values configured in `.env`.

#### Avoid Docker mount errors

If `certs/fleet.crt` or `certs/fleet.key` does not exist when Docker Compose starts, Docker may create a directory instead of a file.

This can produce an error similar to:

```text
not a directory: Are you trying to mount a directory onto a file (or vice-versa)?
```

Always create the certificate files before the first startup.

If the problem has already occurred, stop Fleet and inspect the named volume:

```bash
docker compose stop fleet

docker run --rm \
  -v endpoint-management_data:/data:ro \
  alpine:3.23 \
  sh -c 'ls -ld /data/fleet.crt /data/fleet.key'
```

If both paths are confirmed to be mistakenly created empty directories, remove only those directories:

```bash
docker run --rm \
  -v endpoint-management_data:/data \
  alpine:3.23 \
  sh -c 'rmdir /data/fleet.crt /data/fleet.key'
```

The `rmdir` command removes only empty directories.

Do not delete the entire `endpoint-management_data` volume.

### 5. Start Fleet Server

From the `compose/` directory:

```bash
docker compose up -d
```

Check container status:

```bash
docker compose ps
```

The following services should become healthy:

- `endpoint-management-fleet-1`
- `endpoint-management-mysql-1`
- `endpoint-management-redis-1`

The initialization container, `endpoint-management-fleet-init-1`, is expected to complete successfully and exit.

#### Verify HTTPS

```bash
curl -k -sS -o /dev/null \
  -w 'HTTP %{http_code}\n' \
  https://127.0.0.1:1337/
```

An HTTP `307` redirect can be expected during initial setup.

The `-k` option skips TLS certificate verification because the current certificate is self-signed.

#### Open the web interface

Open:

**https://localhost:1337**

The browser will display a certificate trust warning. This is expected during initial local testing.

Proceed to the Fleet setup page and create the administrator account.

### 6. Local Network Access and Future CA Setup

The current Docker Compose configuration publishes Fleet only on the Mac's loopback interface:

```yaml
ports:
  - "127.0.0.1:${FLEET_SERVER_PORT}:${FLEET_SERVER_PORT}"
```

As a result, Fleet is accessible from the host Mac but **not from other devices on the local network**.

The current self-signed certificate is also valid only for:

- `localhost`
- `127.0.0.1`

These settings are intentional for initial local testing.

#### Future access from Macs and iPhones

To connect other devices on the private local network:

1. Assign the Fleet server a stable local hostname, such as `fleet.home.arpa`.
2. Configure local DNS so client devices can resolve that hostname.
3. Change Docker port publishing to allow connections from the local network.
4. Configure the macOS firewall and network access as required.
5. Create a private local Certificate Authority (CA).
6. Issue a Fleet server certificate signed by that CA, with the server hostname included in the Subject Alternative Name (SAN).
7. Install and trust the public CA certificate on client devices.
8. Replace the temporary Fleet certificate and key with the CA-signed server certificate and its private key.

Only the public CA certificate should be distributed to client devices.

The CA private key must remain secret and must never be committed to Git.

The current self-signed certificate is a temporary solution. Private CA configuration will be implemented in a later stage.

**Note:** TLS trust and Apple MDM enrollment are separate requirements. Apple MDM also requires APNs configuration.

### 7. Stop Fleet Server

From the `compose/` directory:

```bash
docker compose down
```

This stops and removes the Compose containers while preserving named volumes and stored data.

To start Fleet again:

```bash
docker compose up -d
```

Do not use:

```bash
docker compose down -v
```

unless you intentionally want to delete the named volumes, including Fleet and database data.

## Build the macOS fleetd Package

Create a private temporary folder in the repository root, then download and unpack `fleetctl` 4.95.0 there.

```zsh
fleetctl_tmp=$(mktemp -d ./fleetctl-native.XXXXXX) && cd "$fleetctl_tmp"
curl -fLO https://github.com/fleetdm/fleet/releases/download/fleet-v4.95.0/fleetctl_v4.95.0_macos.tar.gz
tar -xzf fleetctl_v4.95.0_macos.tar.gz --strip-components=1 fleetctl_v4.95.0_macos/fleetctl
```

Copy the enrollment secret from **Hosts → Add hosts → macOS** and enter it when prompted.

```zsh
IFS= read -rs 'FLEET_ENROLL_SECRET?Enrollment secret: ' && printf '\n'
```

Build the package from the same folder.

```zsh
arch -arm64 ./fleetctl package \
  --type=pkg --fleet-url=https://localhost:1337 \
  --enroll-secret="$FLEET_ENROLL_SECRET" --fleet-certificate=../compose/certs/fleet.crt \
  --disable-open-folder
```

The output is `fleet-osquery.pkg`. It contains the enrollment secret, so keep it private.

Install it with `sudo installer -pkg ./fleet-osquery.pkg -target /`. Check that the Mac shows **Online** in **Hosts**.
