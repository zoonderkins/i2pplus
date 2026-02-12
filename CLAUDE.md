# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

I2P+ is a soft-fork of the Java I2P anonymous network router. It's a large Java codebase (~200+ Ant build targets) implementing garlic-routed anonymous networking with integrated web applications (torrent client, mail client, HTTP proxy, etc.).

- **Version**: 2.11.0 (core) / 0.9.68 (API) — see `CoreVersion.java` and `RouterVersion.java`
- **License**: AGPL v3
- **Java**: 1.8+ required (some embedded subsystems support 1.6)

## Build Commands

The build system is **Apache Ant**. Prerequisites: JDK 8+, Ant 1.9.8+, GNU gettext (`xgettext`, `msgfmt`, `msgmerge`), UTF-8 locale.

```bash
ant updater              # Dev build: unsigned update zip (fastest for iteration)
ant pkg                  # Full installer package (x86, uses IzPack4)
ant installer-linux      # Linux installer (non-x86)
ant installer-osx        # macOS installer
ant distclean            # Clean all derived files
ant javadoc              # Generate API docs → build/javadoc/index.html
ant i2psnark             # Standalone I2PSnark torrent client
ant zip-linux            # Full install zip (alternative if IzPack fails on Java 14+)
```

Run `ant` with no arguments to see all targets. Create `override.properties` for local build customizations (do not edit `build.properties` directly).

## Testing

```bash
ant test                 # Run all unit tests (core + router + streaming + i2ptunnel)
ant testCore             # Core library tests only
ant testRouter           # Router tests only
ant testStreaming        # Streaming library tests only
ant testI2PTunnel        # I2PTunnel tests only
ant fulltest             # Extended test suite
ant testscripts          # Run shell-based test scripts
```

Tests are JUnit 4 and live under `*/test/junit/` directories mirroring main source structure. Integration tests require a running router with I2CP on `127.0.0.1:7654` (enable via `runIntegrationTests=true` in `override.properties`).

## CI

GitHub Actions (`.github/workflows/ant.yml`): builds updater, installer, javadocs, and I2PSnark on push to master using JDK 8. Also runs CodeQL, PMD, and Google format checks.

## Architecture

Three-layer design:

```
┌─────────────────────────────────────────────┐
│  Applications (apps/)                       │
│  routerconsole, i2psnark, i2ptunnel,        │
│  susimail, susidns, sam, streaming, etc.    │
├─────────────────────────────────────────────┤
│  Router (router/)  →  router.jar            │
│  I2NP protocol, tunnels, transports         │
│  (NTCP/SSU), DHT/Kademlia, peer mgmt       │
├─────────────────────────────────────────────┤
│  Core (core/)  →  i2p.jar                   │
│  Crypto, data structures, I2CP client API,  │
│  utilities, statistics                      │
└─────────────────────────────────────────────┘
```

### Key Modules

| Module | Output | Source |
|--------|--------|--------|
| Core API | `i2p.jar` | `core/java/src/net/i2p/` |
| Router | `router.jar` | `router/java/src/net/i2p/router/` |
| Router Console | webapp | `apps/routerconsole/` (JSP + Java servlets on Jetty) |
| Streaming | `streaming.jar` | `apps/streaming/` (TCP-like over I2P) |
| I2PTunnel | `i2ptunnel.jar` | `apps/i2ptunnel/` (Hidden Service Manager) |
| I2PSnark | `i2psnark.jar` | `apps/i2psnark/` (BitTorrent client) |
| SAM | `sam.jar` | `apps/sam/` (Simple Anonymous Messaging API) |

### Important Subsystems

- **Transport**: `router/.../transport/` — NTCP (TCP-based) and SSU (UDP-based) transports
- **Tunnels**: `router/.../tunnel/` — Garlic routing tunnel construction and management
- **Network DB**: `router/.../networkdb/kademlia/` — Distributed hash table (Kademlia) for router info and leasesets
- **Crypto**: `core/.../crypto/` — ElGamal, DSA, EdDSA, AES, SipHash
- **I2CP**: `core/.../client/` (client-side) + `router/.../client/` (router-side) — Inter-process communication protocol
- **Update system**: `apps/routerconsole/.../update/` — Signed .su3 update packages
- **Native code**: `core/c/` — jbigi (GMP wrapper) and jcpuid (CPU detection)

### Web Console

Jetty-based servlet container serving JSP pages at `http://localhost:7657`. Themes in `apps/routerconsole/jsp/themes/` (classic, dark, light, midnight). The console is the primary UI for router administration, tunnel management, and accessing built-in apps.

## Code Style

- **EditorConfig** is authoritative (`.editorconfig`) — 4-space indent for most Java, with exceptions for vendored code (BOB, susimail, some core libs use tabs)
- CSS files use 2-space indent
- Line endings: LF (except `.bat` files and some vendored code which use CRLF)
- Charset: UTF-8 throughout (except `gnu/getopt/*.properties` which is ISO-8859-1)

## Internationalization

33 languages supported via GNU gettext. `.po`/`.pot` files throughout the source tree. Build requires `gettext` tools. Set `require.gettext=false` in `override.properties` to skip during development.

## Key Ports (when running)

| Port | Protocol | Purpose |
|------|----------|---------|
| 7657 | HTTP | Router console |
| 7667 | HTTPS | Router console (SSL) |
| 4444 | HTTP | HTTP proxy |
| 6668 | IRC | IRC proxy |
| 7654 | I2CP | Client-router protocol |

## Docker

Multi-stage build (Alpine + OpenJDK 21). Quick start:

```bash
cp docker-compose.example.yml docker-compose.yml
docker-compose up --build          # Build and run
# Console at http://127.0.0.1:7667
docker-compose down                # Stop
```

Volumes: `./docker/run/home/config` (router config), `./docker/run/torrents` (I2PSnark). JVM heap default is 512MB (override `JVM_XMX` in `docker/rootfs/startapp.sh`). Uses host networking mode.

## Version Bumping

- Core version: `core/java/src/net/i2p/CoreVersion.java` → `VERSION` and `PUBLISHED_VERSION`
- Router version: `router/java/src/net/i2p/router/RouterVersion.java` → derives from `CoreVersion.VERSION`
