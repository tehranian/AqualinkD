# Contributing to AqualinkD

## Important: Panel Protocol Code is Off-Limits

Do **not** modify code that handles Jandy panel communication protocols (Allbutton, PDA, OneTouch, RSserial, iAqualink, AqualinkTouch). Protocol behavior varies across hardware versions, and changes that work on one panel can break others. If a task touches protocol-level serialization, parsing, or panel state machines, flag it in your PR instead of making changes.

The following files are considered protocol code:
- `allbutton.c/h`, `allbutton_aq_programmer.c`
- `onetouch.c/h`, `onetouch_aq_programmer.c`
- `iaqtouch.c/h`, `iaqtouch_aq_programmer.c`
- `pda.c/h`, `pda_menu.c/h`, `pda_aq_programmer.c`
- `serialadapter.c/h`
- `iaqualink.c/h`
- `aq_serial.c/h` (serial framing and packet I/O)

## Target Platform

AqualinkD is a **Linux-only project** targeting Raspberry Pi (ARM). The C source requires Linux headers (`linux/serial.h`, `systemd/sd-journal.h`). Do not add macOS compatibility stubs or shims. For local development on macOS, use Docker.

## Prerequisites

### Building on Linux (native)

```bash
sudo apt-get install build-essential libsystemd-dev
# For cross-compilation:
sudo apt-get install crossbuild-essential-armhf crossbuild-essential-arm64
sudo apt-get install libsystemd-dev:arm64 libsystemd-dev:armhf
```

### Building on macOS (Docker required)

You need Docker installed. The project provides a Docker-based build environment that handles all Linux cross-compilation toolchains.

First, build the release binary Docker image (one-time setup):

```bash
docker build -f docker/Dockerfile.releaseBinaries -t aqualinkd-releasebin .
```

## Building

All source files are in the `source/` directory. The Makefile compiles to the `build/` directory and outputs binaries to `release/`.

### Make Targets

| Target | Description |
|--------|-------------|
| `make` | Standard build of `aqualinkd` and `rs485mon` (native Linux) |
| `make debug` | Build with debug symbols (`-Wall -O0 -g`) |
| `make aqdebug` | Build with AqualinkD-specific debug flags (`AQ_DEBUG`, `AQ_TM_DEBUG`) |
| `make release` | Build release binaries for ARM via Docker |
| `make quick` | Quick build of arm64 and armhf without clean |
| `make arm64` | Cross-compile for ARM 64-bit |
| `make armhf` | Cross-compile for ARM 32-bit |
| `make amd64` | Cross-compile for x86-64 |
| `make slog` | Build `rs485mon` serial logger only |
| `make runindocker` | Build and run in a Docker container |
| `make clean` | Remove all compiled binaries and object files |

### Typical Workflows

**macOS developer (Docker-based release build):**
```bash
make release
```

**Linux developer (native build):**
```bash
make
# or for debugging:
make debug
```

**Cross-compile for a specific architecture:**
```bash
make arm64
make armhf
```

### Build Flags

The Makefile has configurable feature flags:

```makefile
AQ_PDA = true        # PDA protocol support (default: enabled)
AQ_MANAGER = true    # AQ Manager web UI (requires systemd, default: enabled)
#AQ_RS16 = true      # RS16 panel support (optional)
```

### Compiler Toolchains

| Architecture | Compiler |
|-------------|----------|
| Native (Linux) | `gcc` |
| ARM 64-bit | `aarch64-linux-gnu-gcc` |
| ARM 32-bit | `arm-linux-gnueabihf-gcc` |
| x86-64 | `x86_64-linux-gnu-gcc` |

Libraries linked: `-lpthread -lm -lrt` (plus `-lsystemd` when `AQ_MANAGER` is enabled).

## Output Binaries

After building, binaries are placed in `release/`:

```
release/
├── aqualinkd-arm64      # ARM 64-bit daemon
├── aqualinkd-armhf      # ARM 32-bit daemon
├── rs485mon-arm64       # ARM 64-bit serial logger
├── rs485mon-armhf       # ARM 32-bit serial logger
```

## Code Organization

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for a full module map. Key directories:

```
source/       # All C source and header files (~60 files)
web/          # Web UI static files
release/      # Built binaries, config templates, install scripts
docker/       # Dockerfiles for build and runtime containers
.github/      # CI/CD workflows
```

## Pull Request Guidelines

1. **Do not touch protocol code** (see list above). If your change requires protocol modifications, open an issue to discuss it first.
2. **Test on actual hardware** if possible, or verify via Docker build (`make release`).
3. **Keep changes focused** - one logical change per PR.
4. **Ensure the build succeeds** - at minimum, run `make release` to verify cross-compilation passes.
5. **Follow existing code style** - the codebase uses C99 with 2-space indentation.

## Running Locally (Docker)

For local testing without a Raspberry Pi:

```bash
make runindocker
```

This builds and runs the daemon in a Docker container. It requires a serial device passed through (configured in the Docker compose file at `docker/docker-compose.yml`).

## Installation on Raspberry Pi

The project provides install scripts in `release/`:

```bash
# Remote install (latest release):
curl -fsSL https://install.aqualinkd.com | sudo bash -s -- latest

# Local install (from built binaries):
sudo ./release/install.sh
```

Installation places:
- Binary: `/usr/local/bin/aqualinkd`
- Config: `/etc/aqualinkd.conf`
- Service: `/etc/systemd/system/aqualinkd.service`
- Web UI: `/var/www/aqualinkd/`

## Resources

- [Project Wiki](https://github.com/aqualinkd/AqualinkD/wiki)
- [API Reference](docs/API.md)
- [Architecture Overview](docs/ARCHITECTURE.md)
