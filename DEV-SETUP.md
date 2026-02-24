# Development Setup

## Docker-based Development Environment

### Prerequisites

- Docker installed and accessible by your user

### Build the Ubuntu 24.04 CI Image

This image compiles FRR from source with many features enabled (gRPC, RPKI, SNMP, scripting, sharpd, gcov, protobuf).

From the repo root:

```bash
docker build -t frr-ubuntu24:latest --build-arg=UBUNTU_VERSION=24.04 -f docker/ubuntu-ci/Dockerfile .
```

### Run the Container

```bash
docker run -d --init --privileged --name frr-ubuntu24 \
    --mount type=bind,source=/lib/modules,target=/lib/modules \
    frr-ubuntu24:latest
```

### Interactive Shell

```bash
docker exec -it frr-ubuntu24 bash
```

### Run `make check` (Unit Tests)

```bash
docker exec frr-ubuntu24 bash -c 'cd ~/frr ; make check'
```

### Stop and Remove Container

```bash
docker stop frr-ubuntu24 ; docker rm frr-ubuntu24
```

### Remove Image

```bash
docker rmi frr-ubuntu24:latest
```

### Build the Topotests Image

This image provides the environment for running integration (topology) tests.

From the repo root:

```bash
cd tests/topotests && docker build --pull -t frrouting/topotests:latest .
```

Or use the provided script:

```bash
tests/topotests/docker/build.sh
```

## Other Ubuntu Versions

Replace `UBUNTU_VERSION=24.04` with `22.04` or `20.04`. Each version has its own README under `docker/ubuntu{version}-ci/`.
