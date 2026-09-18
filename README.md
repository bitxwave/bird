# bird

Docker image and config for [BIRD](https://bird.network.cz/) (BIRD Internet Routing Daemon).

### Overview

This repo provides a minimal Docker image for **BIRD**, the widely used BGP/OSPF routing daemon. The image is based on Debian Trixie slim and runs BIRD as a non-root user.

### Features

- **Multi-version**: Build any BIRD release (e.g. 2.0.10, 3.0.1) via build-arg — sources come from the official release tarball, falling back to the [upstream git archive](https://gitlab.nic.cz/labs/bird) when it is unreachable.
- **Multi-arch**: Image is built for `linux/amd64` and `linux/arm64`; the correct variant is pulled automatically.
- **Slim**: Multi-stage build; only runtime deps in the final image.
- **Non-root**: Runs as user `bird`; the `bird` binary has file capabilities (`cap_net_raw`, `cap_net_admin`) so OSPF/BGP work without root.
- **Config-friendly**: Mount your own `bird.conf`; if missing, the example config is used with stderr logging appended so `docker logs` works.
- **Healthcheck**: Built-in healthcheck using `birdc show status`.

### Image

Images are published to GitHub Container Registry. Use a version tag that matches an official BIRD release (recommended) or `latest` (last build from a tag).

```sh
# Pull by version (recommended; matches BIRD release)
docker pull ghcr.io/picopock/bird:2.0.10
docker pull ghcr.io/picopock/bird:3.0.1

# Or pull latest (most recent tagged build)
docker pull ghcr.io/picopock/bird:latest
```

### Quick start

```sh
# Run with your config (recommended: host network so neighbors can discover this router)
docker run -d --name bird \
  --network host \
  --cap-add=NET_ADMIN --cap-add=NET_RAW \
  -v /path/to/your/bird.conf:/etc/bird/bird.conf:ro \
  ghcr.io/picopock/bird:latest

# Run with default config (example config only; neighbor discovery may need --network host)
docker run -d --name bird \
  --cap-add=NET_ADMIN --cap-add=NET_RAW \
  ghcr.io/picopock/bird:latest
```

**Network and neighbor discovery:** For other routers to discover this instance, use either `--network host` (BIRD uses the host’s interfaces; recommended for OSPF/BGP) or a macvlan on the same segment and peer with the container’s IP. With default bridge networking, OSPF multicast and BGP (port 179) are not reachable from the LAN.

**Capabilities:** The image sets file capabilities on the `bird` binary so it can use raw sockets as non-root. Still use `--cap-add=NET_ADMIN --cap-add=NET_RAW` when running the container so OSPF neighbors establish correctly in all environments.

### Configuration

- **Config file**: `/etc/bird/bird.conf`
- **Control socket**: `/var/run/bird/bird.ctl`
- **Example config**: `/etc/bird/bird.conf.example`

Volumes:

- `/etc/bird` — config directory (mount your `bird.conf` here if needed).
- `/var/run/bird` — runtime (control socket, etc.).

If `/etc/bird/bird.conf` is missing at startup, the entrypoint copies the example config and appends `log stderr all;`. The control socket path is passed to BIRD with `-s`, not set in the config file.

### Building locally

```sh
# Default build (BIRD 3.0.1)
docker build -t bird:local .

# Build a specific BIRD version
docker build -t bird:2.0.10 --build-arg BIRD_VERSION=2.0.10 .
```

Tag must be a valid semver `x.y.z` matching an upstream BIRD release tag (see [release tags](https://gitlab.nic.cz/labs/bird/-/tags)).

### Versions and CI

- Pushing a **tag** like `v2.0.10` or `v3.0.1` triggers the GitHub Actions workflow.
- The workflow builds the image with `BIRD_VERSION` set from the tag (e.g. `v2.0.10` → `2.0.10`) and pushes to `ghcr.io/picopock/bird` with tags `2.0.10` and `latest`.
- Tag format must be **semver** (e.g. `v1.2.3`); otherwise the workflow fails.

### License

BIRD is licensed under the GNU General Public License. See [BIRD project](https://bird.network.cz/) for details. This Docker image and config are provided as-is.
