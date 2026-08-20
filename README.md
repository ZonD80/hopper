# ɹǝddoH (Hopper)

<p align="center">
  <img src="screenshots/app-icon.png" width="128" alt="ɹǝddoH app icon">
</p>

iOS and Android VPN clients plus a Linux server stack for a **multi-hop SSH overlay**: traffic is tunneled as raw IP packets over SSH, routed through a chain of nodes, and NAT’d at the exit. One app, one server daemon (`hopperd`), no legacy relays.

## Getting the app

| Platform | Status |
| -------- | ------ |
| **iOS (TestFlight)** | [Join the beta](https://testflight.apple.com/join/QCeqF71s) — install [TestFlight](https://apps.apple.com/app/testflight/id899247664) first, then open the link on your iPhone or iPad |
| **iOS / Android (developer-signed)** | Available in [GitHub Releases](https://github.com/ZonD80/hopper/releases) — install the IPA or APK from the latest release |
| **Apple App Store** | [Download on the App Store](https://apps.apple.com/us/app/%C9%B9%C7%9Dddoh/id6778506930) |
| **iOS (appdb)** | <a href="https://appdb.to/details/aa708e634598470eeec6ebd64cdd5fda9a6f5dc1"><img title="Get from appdb" src="https://s3cdn.dbservices.to/official_buttons/get_white.png" width="100" alt="Get from appdb" /></a> |
| **Google Play** | [Get it on Google Play](https://play.google.com/store/apps/details?id=com.aengix.hopper) |

For iOS, use the [App Store](https://apps.apple.com/us/app/%C9%B9%C7%9Dddoh/id6778506930), [appdb](https://appdb.to/details/aa708e634598470eeec6ebd64cdd5fda9a6f5dc1), or the TestFlight beta above. For Android, use Google Play. Alternatively, use the signed builds from Releases (you may need to trust the developer certificate on iOS or allow installs from unknown sources on Android).

## Setting up a server

Each hop in your chain runs `hopperd` on a Linux VPS. You need one VPS per hop (a single VPS works for a one-hop chain where entry = exit).

### Get a VPS

For the simplest setup, rent the **cheapest Ubuntu VPS** you can find (any provider — Hetzner, DigitalOcean, Vultr, etc.). Use a fresh machine with:

- **User:** `root`
- **Auth:** root password (SSH password login enabled)

The server must have TUN support (`/dev/net/tun`), `python3`, `ip`, and `iptables` (for exit NAT). Ubuntu images from major providers include these by default.

### Deploy from the app

You do **not** need a Mac or `deploy.sh` to get started. The app installs and configures the server for you:

1. Open **ɹǝddoH** → **Configure chains** → **Server library**.
2. Tap **Deploy**.
3. Enter the VPS **host/IP**, **user** (`root`), **SSH port** (`22`), and **root password**.
4. Wait for the deploy log to finish — the server is added to your library automatically (no QR scan needed).

The app uploads `hopperd`, runs `configure_server.sh`, generates a deploy key, and saves the server profile locally. Repeat for each hop.

**Alternative (developers):** deploy from your Mac with `./deploy.sh` — see [Deploy from the command line](#deploy-from-the-command-line).

You can also add servers via **Scan QR** or **Import** (`.hopperconf` file / paste JSON) if you used the CLI deploy flow.

## Screenshots

### iPhone

<p align="center">
  <img src="screenshots/iphone/01-home.png" width="200" alt="Home — chain, connect, and route">
  <img src="screenshots/iphone/02-chain.png" width="200" alt="Chain detail — entry to exit hops">
  <img src="screenshots/iphone/03-servers.png" width="200" alt="Server library">
  <img src="screenshots/iphone/04-provisioning.png" width="200" alt="Provisioning chain on connect">
</p>

### iPad

<p align="center">
  <img src="screenshots/ipad/01-home.png" width="280" alt="Home — connected VPN">
  <img src="screenshots/ipad/02-servers.png" width="280" alt="Server library">
  <img src="screenshots/ipad/03-chain.png" width="280" alt="Chain detail">
  <img src="screenshots/ipad/04-chains.png" width="280" alt="Chains list">
</p>

### Android (phone)

<p align="center">
  <img src="screenshots/android/phone/01-home.png" width="200" alt="Home — chain, connect, and route">
  <img src="screenshots/android/phone/02-chains.png" width="200" alt="Chains list">
  <img src="screenshots/android/phone/03-chain-detail.png" width="200" alt="Chain detail — entry to exit hops">
  <img src="screenshots/android/phone/04-servers.png" width="200" alt="Server library">
  <img src="screenshots/android/phone/05-server-detail.png" width="200" alt="Server detail">
</p>

---

## How it works

```mermaid
flowchart LR
  subgraph clients [Mobile clients]
    App[iOS / Android app]
    Tunnel[Packet tunnel]
    App -->|provision chain| Tunnel
    Tunnel -->|SSH + iptunnel| Entry
  end

  subgraph chain [Server chain entry to exit]
    Entry[hopperd entry]
    Relay[hopperd relay]
    Exit[hopperd exit]
    Entry -->|SSH pipe| Relay
    Relay -->|SSH pipe| Exit
  end

  Tunnel --> Entry
  Exit -->|TUN + NAT| Internet[(Internet)]
```

| Layer           | Role                                                                                             |
| --------------- | ------------------------------------------------------------------------------------------------ |
| **iOS**         | L3 VPN (`NEPacketTunnelProvider`). All IPv4 default traffic → leased overlay client address.     |
| **Android**     | L3 VPN (`VpnService`). Same overlay address and iptunnel data plane as iOS.                      |
| **iptunnel** | Framed IP over a byte stream (SSH `direct-tcpip` to `127.0.0.1:7400`).                           |
| **hopperd**  | Userspace routing between ingress (client), `next` (downstream hop), and TUN (internet on exit). |
| **SSH**      | App → entry hop; each hop → next hop via local `~/.hopper/id_ed25519`.                           |

### Two-phase connect

1. **Provision (exit → entry)** — The app SSH-execs `start_server.sh` on each hop, last to first: trust upstream keys, write `hopper.json`, start `hopperd`.
2. **VPN (entry only)** — Extension SSH-connects to the **first** hop in the chain, opens iptunnel to local `hopperd`, and carries packets.

Chain order in the app: **first = entry**, **last = exit**.

### Multi-chain and multi-device

Hopper supports **multiple chains** and **multiple clients per chain** on the same servers.

**Multi-chain** — Each chain has its own UUID. That ID derives a dedicated overlay subnet (`10.64.{octet}.0/24`), local `hopperd` listen port (`7400 + octet`), and TUN interface (`hopper_*`) on every hop. State lives under `~/.hopper/chains/{chain_id}/`; active chains are tracked in `~/.hopper/registry.json`. The same VPS can serve several chains at once (as entry, relay, or exit in different chains).

**Multi-device** — Several phones or tablets can connect to the **same chain** at the same time. Each app install gets a stable device ID; the entry hop assigns a unique client address from that chain's pool (`10.64.{octet}.2`–`.254`) via lease. Leases renew while connected and expire after idle timeout (default 1 hour).

**TUN limits** — Each active chain uses **one TUN interface per hop** where `hopperd` runs. How many chains you can run in parallel on a server depends on how many TUN devices the host allows (typically many on a stock Linux VPS, but the limit varies by kernel and provider).

### Overlay (per chain)

Each chain gets its own `/24` subnet: `10.64.{octet}.0/24`, where `{octet}` is derived from the chain UUID (1–254).

| Address | Use |
| ------- | --- |
| `10.64.{octet}.2`–`.254` | Mobile clients (leased per device) |
| `10.64.{octet}.10` + index | Hop *i* in chain (entry = `.10`) |
| `0.0.0.0/0` | Relay → `next`; exit → TUN + NAT |

---

## Requirements

### Server (each hop)

- Linux with TUN (`/dev/net/tun`)
- `python3`, `ip`, `iptables` (exit NAT)
- Root or `cap_net_admin` on `hopperd` (set by `configure_server.sh` when run as root)
- SSH access for deploy and for inter-hop / client connections

### iOS

- Xcode 16+, iOS 17+
- Apple Developer account with **Network Extension** (Packet Tunnel) entitlement
- App Group: `group.com.aengix.hopper`

### Android

- Android Studio with SDK 35 (API 26+ devices)
- JDK 17 (bundled with Android Studio on macOS)
- Release signing: `~/googlePlayKeys.jks` + `app-android/keystore.properties` (see [Android build & run](#build--run))

### Dev machine

- Go 1.22+ (build server binaries)
- SSH key to target servers (`~/.ssh/id_rsa` by default)

---

## Quick start

1. [Get the app](#getting-the-app) and [set up one or more servers](#setting-up-a-server).
2. **New chain** → name it → **Add server…** in order **entry → exit** (pick servers from the library).
3. Swipe **Use** (iOS) or tap **Use** (Android), or pick the chain on the home screen → **Connect**.

### Remove a hop

```bash
./remove.sh YOUR_SERVER_IP
```

Stops `hopperd`, removes `~/hopper`, `~/.hopper`, TUN `hopper0`, hopper NAT rules, and hopper lines in `authorized_keys`.

---

## Server reference

### Deploy from the command line

From your Mac:

```bash
cd server
./deploy.sh YOUR_SERVER_IP
```

This will:

- Build `dist/hopperd-linux-{amd64,arm64}`
- Upload bundle to `~/hopper` on the server
- Run `configure_server.sh --json-only` and open a **local QR page** in the browser (deleted after 5 seconds)

Options (same as `remove.sh`):

| Flag         | Default         | Meaning              |
| ------------ | --------------- | -------------------- |
| `-u`         | `root`          | SSH user             |
| `-p`         | `22`            | SSH port             |
| `-i`         | `~/.ssh/id_rsa` | SSH private key      |
| `-P`         | `~/hopper`      | Remote install path  |
| `-y`         | —               | Skip confirmation    |
| `--no-build` | —               | Skip `build_dist.sh` |

Environment: `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_PORT`, `DEPLOY_KEY`, `DEPLOY_PATH`.

### Layout on the machine

```
~/hopper/
  configure_server.sh   # one-time / re-run: keys + JSON profile
  start_server.sh       # app-invoked: config + start hopperd
  hopper_common.sh
  dist/
    hopperd-linux-amd64
    hopperd-linux-arm64

~/.hopper/
  id_ed25519            # inter-hop + hopperd SSH identity
  registry.json         # active chains
  chains/{chain_id}/
    hopper.json         # runtime config for this chain
    hopper-ready        # READY port line while running
    hopper-YYYY-MM-DD.log  # daily hopperd logs (default retain 2 days)
```

### Scripts

| Script                | Who runs it         | Purpose                                                                          |
| --------------------- | ------------------- | -------------------------------------------------------------------------------- |
| `configure_server.sh` | Admin / `deploy.sh` | Generate host keypair, `authorized_keys`, optional `setcap`, emit QR JSON        |
| `start_server.sh`     | Mobile app via SSH exec | `--stop-only`, `--trust-pubkey`, write config, start `hopperd`, print ready JSON |
| `deploy.sh`           | Developer           | Build, upload, configure, browser QR                                             |
| `remove.sh`           | Developer           | Uninstall                                                                        |
| `build_dist.sh`       | Developer           | Cross-compile `hopperd`                                                          |

#### `configure_server.sh`

```bash
./configure_server.sh                    # interactive on server
./configure_server.sh --json-only --host 1.2.3.4 --port 22
```

#### `start_server.sh` (provision)

```bash
./start_server.sh --role exit --addr 10.64.0.12 --index 2 \
  --overlay 10.64.0.0/24 --client-addr 10.64.0.2

./start_server.sh --role relay --addr 10.64.0.11 --index 1 \
  --client-addr 10.64.0.2 \
  --next-host hop2.example.com --next-port 22 --next-user root
```

Stdout: one JSON line, e.g. `{"ready":true,"mode":"exit","addr":"10.64.0.12",...}`.

### `hopperd`

```bash
# typical chain paths (created by hopper start / the app)
CHAIN=~/.hopper/chains/<chain_id>
./dist/hopperd-linux-amd64 -verbose \
  --config "$CHAIN/hopper.json" \
  --ready-file "$CHAIN/hopper-ready" \
  --log-dir "$CHAIN" \
  --log-keep-days 2
```

| Flag | Default | Purpose |
| --- | --- | --- |
| `--config` | `~/.hopper/hopper.json` | Runtime JSON config |
| `--ready-file` | — | Write `READY <port>` when listening |
| `--log-dir` | — | Daily logs as `hopper-YYYY-MM-DD.log` in this directory; rotate at local midnight; prune on write and hourly |
| `--log-keep-days` | `2` | Calendar days of logs to keep (today counts as 1; `2` = today + yesterday) |
| `-verbose` | off | Debug logging |

When started via `hopper start` / the app, Python passes `--log-dir` and `--log-keep-days`. Override retention with env `HOPPER_LOG_KEEP_DAYS` (same default `2`).

Listens on `127.0.0.1:7400` (or `7400 + overlay octet` per chain) only — reached via SSH forwarding.

Example config: [`server/hopper.example.json`](server/hopper.example.json).

### Build server only

```bash
cd server
./build_dist.sh
```

Binaries land in `server/dist/` (gitignored).

---

## iOS app reference

### Screens

| Screen               | Purpose                                         |
| -------------------- | ----------------------------------------------- |
| **Home**             | Select chain, connect/disconnect, route preview |
| **Configure chains** | Create/delete chains, open server library       |
| **Chain detail**     | Name, reorder hops, add/remove servers          |
| **Server library**   | **Deploy** new servers, scan QR, import `.hopperconf` / JSON, delete servers |
| **Export**           | QR (in-person) + encrypted `.hopperconf` share for servers and chains |

Profiles persist in the App Group (`hopper-profiles.json`).

### Server profile JSON (v2)

Scan QR, **Import**, and `./deploy.sh` all use the same **v2** hop wire format (`HopProfileCodec` on iOS/Android, `server/hopper/node_profile.py` on the server). Encrypted cross-device shares use [`.hopperconf`](#hopperconf-share-files-v1). The app stores servers in a library; chains reference server IDs in order.

**Example** (as emitted by `configure_server.sh --json-only`):

```json
{
  "v": 2,
  "name": "vps.example.com",
  "host": "203.0.113.10",
  "port": "22",
  "user": "root",
  "private_key": "-----BEGIN OPENSSH PRIVATE KEY-----\n...",
  "install_dir": "~/hopper",
  "server_version": "2.0.1",
  "min_app_version": "2.0.0",
  "host_key": ["ssh-ed25519 AAAA..."]
}
```

| Field | Required | Notes |
| ----- | -------- | ----- |
| `v` | No | Must be `2` when present; other values are rejected |
| `host` | Yes | SSH hostname or IP. Alias: `server` |
| `user` | Yes | SSH user. Alias: `username` |
| `private_key` | Yes | OpenSSH private key for this hop (`~/.hopper/id_ed25519` from configure). Alias: `privateKey` |
| `port` | No | SSH port as a string; default `22` |
| `name` | No | Display label; defaults to host or `Untitled`. Aliases: `title`, `remarks` |
| `install_dir` | No | Remote bundle path; default `~/hopper`. Aliases: `installDir`, `hopper_dir` |
| `server_version` | No | Server bundle version from `VERSION.json`; export uses `unknown` if empty. Alias: `serverVersion` |
| `min_app_version` | No | Minimum client version required by the server; export uses `unknown` if empty. Alias: `minAppVersion` |
| `host_key` | No | SSH host key pin(s) for first connect — string or array. Alias: `hostKeys` |

Treat exported JSON and QR codes as **secrets** (they contain the hop private key).

### `.hopperconf` share files (v1)

Servers and chains are shared between devices as encrypted **`.hopperconf`** files (`application/x-hopperconf`). QR codes stay **unencrypted** for in-person scanning only — they are never written to a shareable file.

#### Envelope (on disk)

Always JSON. Private keys live only inside the encrypted `data` blob.

```json
{
  "v": 1,
  "fmt": "hopperconf",
  "alg": "aes-256-gcm",
  "kdf": "pbkdf2-sha256",
  "iter": 210000,
  "salt": "<base64 16 bytes>",
  "nonce": "<base64 12 bytes>",
  "data": "<base64 ciphertext || 16-byte GCM tag>"
}
```

| Field | Notes |
| ----- | ----- |
| `fmt` | Must be `hopperconf` |
| `alg` | `aes-256-gcm` (AES-256-GCM, 128-bit tag appended to ciphertext) |
| `kdf` | `pbkdf2-sha256` — PBKDF2-HMAC-SHA256 over the **UTF-8** password |
| `iter` | Iteration count (apps use `210000`) |
| `salt` / `nonce` | Random per file (`16` / `12` bytes) |
| `data` | `AES-GCM(key, nonce, plaintext)` output: ciphertext ‖ tag |

**Password:** optional at export. Empty / omitted at export → default password `ɹǝddoH` (app display name). On import, apps try the **default password first**, then a user-provided password if that fails. Derived key length is 32 bytes.

#### Plaintext payload (after decrypt, or QR)

**Server** (file payload uses `kind`; QR for a single server may still be bare hop-profile v2):

```json
{
  "v": 1,
  "kind": "server",
  "server": { "v": 2, "name": "...", "host": "...", "port": "22", "user": "...", "private_key": "...", "...": "..." }
}
```

**Chain** (embeds full hop profiles in order; import mints new server IDs and a new chain):

```json
{
  "v": 1,
  "kind": "chain",
  "name": "My chain",
  "hops": [ { "v": 2, "...": "..." }, { "v": 2, "...": "..." } ]
}
```

Apps open `.hopperconf` via the share sheet / Files / “open with”. Interop tests (Python reference, Swift CryptoKit, Android `HopperConf`) live under `tests/hopperconf/` — run `./tests/hopperconf/run.sh`.

### Build & run

```bash
cd app
open Hopper.xcodeproj
```

- Target **Hopper** (app) + **HopperExtension** (packet tunnel)
- Citadel (vendored SSH) is linked to both app (provision) and extension (data plane)
- Signing: set your `DEVELOPMENT_TEAM`, enable Network Extension + App Groups

Regenerate Xcode project (optional):

```bash
ruby app/Scripts/generate_xcodeproj.rb
```

### Project layout

```
app/
  Hopper/              SwiftUI app, VPNController, ChainProvisioner
  HopperExtension/     PacketTunnelProvider
  TunnelCore/          SSHHopConnector, IPTunnelEngine, HopSSH
  Shared/              Models, HopConstants, ProfileStore
  Vendor/Citadel/      SSH client library
app-android/
  app/                 Jetpack Compose UI, VpnController, HopperVpnService
  build-apk.sh         Signed release APK (bumps versionCode)
  build-aab.sh         Signed Play Store bundle (bumps versionCode)
  generate-keystore.sh Create hopper-upload signing key
server/
  cmd/hopperd/         Daemon entrypoint
  internal/hopper/     Config, session routing, NAT, SSH next-hop
  internal/iptunnel/   Frame protocol + Linux TUN
```

---

## Android app reference

Same screens and flow as iOS: home (chain + connect), chain configurator, chain detail, server library (**Deploy**, QR scan + `.hopperconf` / JSON import), encrypted file share for servers and chains.

Profiles persist in app-private storage (`hopper-profiles.json`). Server profile JSON and `.hopperconf` formats are identical to iOS — see [Server profile JSON (v2)](#server-profile-json-v2) and [`.hopperconf` share files (v1)](#hopperconf-share-files-v1).

### Build & run

Open the project in Android Studio:

```bash
cd app-android
open -a "Android Studio" .
```

Or build a debug APK from the command line:

```bash
cd app-android
./gradlew assembleDebug
```

Output: `app/build/outputs/apk/debug/app-debug.apk`

Install on a connected device:

```bash
adb install app/build/outputs/apk/debug/app-debug.apk
```

On first **Connect**, Android prompts for VPN permission — required for the tunnel.

### Release signing

Release builds use the same keystore layout as other AENGIX Android apps. The keystore lives outside the repo:

```
~/googlePlayKeys.jks
```

Create `app-android/keystore.properties` (gitignored) before running the release scripts:

```properties
storePassword=your_store_password
keyPassword=your_key_password
keyAlias=hopper-upload
```

Hopper uses alias `hopper-upload` in the shared AENGIX keystore (`upload` is reserved for the TV browser app).

To generate a new Hopper signing key (adds to existing keystore or creates a fresh one):

```bash
cd app-android
./generate-keystore.sh
```

Or manually:

```bash
keytool -genkeypair -v \
  -keystore ~/googlePlayKeys.jks \
  -alias hopper-upload \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -storepass YOUR_STORE_PASSWORD \
  -keypass YOUR_KEY_PASSWORD \
  -dname "CN=Hopper, OU=Mobile, O=AENGIX SL, L=Barcelona, ST=Barcelona, C=ES"
```

### Release build scripts

Both scripts auto-detect the Android Studio JDK, validate signing credentials, **bump `versionCode` by 1**, and produce a signed artifact:

```bash
cd app-android
./build-apk.sh    # app/build/outputs/apk/release/app-release.apk
./build-aab.sh    # app/build/outputs/bundle/release/app-release.aab
```

For manual Gradle release builds (without auto bump):

```bash
cd app-android
./gradlew assembleRelease   # APK
./gradlew bundleRelease     # AAB
```

### Android project layout

```
app-android/
  app/src/main/java/com/aengix/hopper/
    ui/                  Compose screens
    vpn/                 VpnController, HopperVpnService, TunnelCoordinator
    ssh/                 HopSSH, SSHHopConnector (SSHJ)
    tunnel/              IPTunnelFrame, IPTunnelEngine
    provision/           ChainProvisioner
    model/               AppState, HopNodeProfile, HopChain
    data/                ProfileStore, HopQRParser
```

---

## Troubleshooting

| Symptom                   | Things to check                                                                  |
| ------------------------- | -------------------------------------------------------------------------------- |
| VPN connects, no internet | Exit NAT: `iptables -t nat -L`; today’s `hopper-YYYY-MM-DD.log` on exit; re-connect to re-provision |
| Chain provision fails     | SSH from app to each hop; `start_server.sh` on server; keys in `authorized_keys` |
| `hopperd` won’t start     | Root/`setcap cap_net_admin`; read `~/.hopper/chains/<chain_id>/hopper-YYYY-MM-DD.log` |
| Extension / VPN errors  | iOS: App Group + embedded extension; reinstall VPN profile. Android: revoke/re-grant VPN permission; check logcat `Hopper` |

**Logs**

- Server: `~/.hopper/chains/<chain_id>/hopper-YYYY-MM-DD.log` (daily; default keep 2 days via `--log-keep-days` / `HOPPER_LOG_KEEP_DAYS`)
- iOS: Xcode → Window → Devices → open console for device, filter `Hopper`
- Android: `adb logcat -s Hopper`

**Manual stop on server**

```bash
cd ~/hopper && ./start_server.sh --stop-only
```

---

## Security notes

- QR and deploy HTML contain **private keys** — treat as secrets; deploy deletes local HTML after 5s.
- Each hop has its own `~/.hopper/id_ed25519`; provision adds upstream pubkeys to downstream `authorized_keys`.
- `hopperd` binds to loopback; only SSH-forwarded clients reach iptunnel.
- Review `authorized_keys` after `remove.sh` if you added keys manually.

---

## License

See repository for license terms. Citadel is vendored under its own license in `app/Vendor/Citadel/`.
