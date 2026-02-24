# Development Testing

## Unit Tests

### Inside the Ubuntu 24.04 CI Container

```bash
docker exec frr-ubuntu24 bash -c 'cd ~/frr ; make check'
```

## Topotests (Integration Tests)

Topotests use pytest and run FRR daemons in network namespaces to test real routing behavior. They require the `frrouting/topotests:latest` Docker image (see [DEV-SETUP.md](DEV-SETUP.md)).

### Run a Specific Test

```bash
./tests/topotests/docker/frr-topotests.sh bgp_l3vpn_to_bgp_vrf/test_bgp_l3vpn_to_bgp_vrf.py
```

### Run a Specific Test with Verbose Output

```bash
./tests/topotests/docker/frr-topotests.sh -vv -s bgp_l3vpn_to_bgp_vrf/test_bgp_l3vpn_to_bgp_vrf.py
```

### Run All Topotests

```bash
./tests/topotests/docker/frr-topotests.sh
```

Or using make:

```bash
make topotests
```

### Drop into a Shell (Build Only, No Tests)

```bash
./tests/topotests/docker/frr-topotests.sh /bin/bash
```

### Force a Clean Rebuild

```bash
TOPOTEST_CLEAN=1 ./tests/topotests/docker/frr-topotests.sh
```

### Run Full Topotests Directly in the Ubuntu 24.04 CI Container

```bash
docker run --init -it --privileged --name frr-ubuntu24 \
    -v /lib/modules:/lib/modules frr-ubuntu24:latest \
    bash -c 'cd /home/frr/frr/tests/topotests; sudo pytest -nauto --dist=loadfile'
```

### Run a Specific Test in the Ubuntu 24.04 CI Container

```bash
docker exec frr-ubuntu24 bash -c \
    'cd ~/frr/tests/topotests ; sudo pytest bgp_l3vpn_to_bgp_vrf/test_bgp_l3vpn_to_bgp_vrf.py'
```

## Analyzing Test Results

### Extract Results from a Stopped Container

```bash
tests/topotests/analyze.py -C frr-ubuntu24 -Ar run-results
```

### Extract Coverage from a Stopped Container

```bash
docker export frr-ubuntu24 | tar --strip=3 --wildcards -vx '*.gc??'
lcov -b $(pwd) --capture --directory . --output-file=coverage.info
```

## Environment Variables

| Variable | Description |
|---|---|
| `TOPOTEST_CLEAN` | Set to `1` to force a full rebuild before testing |
| `TOPOTEST_VERBOSE` | Show detailed build output (enabled by default) |
| `TOPOTEST_SANITIZER` | Address sanitizer (enabled by default, set `0` to disable) |
| `TOPOTEST_LOGS` | Custom log directory (default: `/tmp/topotest_logs`) |
| `TOPOTEST_FRR` | Path to FRR tree (default: current git repo) |
| `TOPOTEST_AUTOLOAD` | Set to `1` to auto-load kernel modules without prompting |
| `TOPOTEST_NOLOAD` | Set to `1` to skip kernel module loading entirely |
