# fleet-arm64

Build and run Fleet Server 4.95.0 locally on Apple Silicon. The Docker Compose stack runs Fleet natively on ARM64 alongside MySQL and Redis. Fleet listens on `https://localhost:1337` on the host Mac when using the default settings.

## Build Fleet Server

Use an Apple Silicon Mac with OrbStack or Docker Desktop, Docker Compose, Docker Buildx, Git, and OpenSSL. Run these commands from the repository root.

```bash
git clone --branch fleet-v4.95.0 --depth 1 \
  https://github.com/fleetdm/fleet.git src/fleet-arm64

docker buildx build --platform linux/arm64 \
  --file compose/Dockerfile.fleet-arm64 \
  --tag fleet-local:4.95.0-arm64 --load src/fleet-arm64
```

If the Fleet source is already checked out at `src/fleet-arm64`, start with the build command.

## Configure and start Fleet

Enter `compose/` and copy the example on the first setup.

```bash
cd compose
cp .env.example .env
```

Set `MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD`, and `FLEET_SERVER_PRIVATE_KEY` in `.env`. Use `openssl rand -hex 32` to generate the Fleet server key.

Generate the local TLS certificate before the first start. If you already have `certs/fleet.crt` and `certs/fleet.key`, reuse them so enrolled agents continue to trust the server.

```bash
mkdir -p certs
openssl req -x509 -newkey rsa:3072 -sha256 -nodes -days 365 \
  -keyout certs/fleet.key -out certs/fleet.crt \
  -subj '/CN=localhost' \
  -addext 'subjectAltName=DNS:localhost,IP:127.0.0.1'
chmod 600 certs/fleet.key
chmod 644 certs/fleet.crt
```

```bash
docker compose up -d
docker compose ps
curl -fsS --cacert certs/fleet.crt https://localhost:1337/healthz
```

Once Fleet is healthy, open <https://localhost:1337> and create the administrator account. The browser may ask you to trust the self-signed certificate. Compose retains the `endpoint-management` project name so existing containers and named volumes still belong to the same stack. Run `docker compose down` from `compose/` to stop the stack and keep its data.

## Install the macOS agent

On the Mac running Fleet, return to the repository root. Download the matching `fleetctl` release and get the enrollment secret from **Hosts → Add hosts → macOS** in Fleet.

```bash
fleetctl_dir=$(mktemp -d "$PWD/fleetctl-native.XXXXXX")
curl -fL -o "$fleetctl_dir/fleetctl.tar.gz" \
  https://github.com/fleetdm/fleet/releases/download/fleet-v4.95.0/fleetctl_v4.95.0_macos.tar.gz
tar -xzf "$fleetctl_dir/fleetctl.tar.gz" -C "$fleetctl_dir" \
  --strip-components=1 fleetctl_v4.95.0_macos/fleetctl

printf 'Enrollment secret: '
IFS= read -rs fleet_enroll_secret
printf '\n'
(
  cd "$fleetctl_dir" || exit 1
  arch -arm64 ./fleetctl package --type=pkg --fleet-url=https://localhost:1337 \
    --enroll-secret="$fleet_enroll_secret" --fleet-certificate=../compose/certs/fleet.crt \
    --disable-open-folder
)
unset fleet_enroll_secret
```

```bash
sudo installer -pkg "$fleetctl_dir/fleet-osquery.pkg" -target /
```

Check that the Mac shows **Online** in Fleet. The `.pkg` contains the enrollment secret. Git ignores the package, TLS files, and `.env`.
