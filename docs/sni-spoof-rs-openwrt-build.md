# Building `sni-spoof-rs` for OpenWrt — Native Alpine Build

This guide compiles `sni-spoof-rs` on an **Alpine Linux machine or VM** using
only Alpine's own packages. No Docker, no `cross`, no musl.cc download, no
external toolchains. Alpine already uses musl libc natively, so every binary
it produces is statically linkable without any extra setup.

If you do not have an Alpine machine, the quickest way to get one is a cheap
VPS, a local VM (VirtualBox/QEMU), or a spare machine booted from the Alpine
ISO. A 1 GB RAM / 4 GB disk instance is more than enough.

---

## Why Alpine?

Alpine's system compiler targets `musl` by default. When you install
`gcc` and `musl-dev` on Alpine and compile a Rust project, the result is
already a musl-linked binary — static by default with no linker flags needed.
This is exactly what OpenWrt requires: a self-contained binary with no shared
library dependencies.

For **native architecture** (building aarch64 on an aarch64 Alpine host, etc.)
you need nothing beyond the standard Alpine packages. For **cross-compilation**
from x86-64 to MIPS or ARM, Alpine ships pre-built cross toolchains in its
community repository.

---

## Step 0 — Identify Your Router's Architecture

SSH into the router:

```sh
uname -m
```

| `uname -m` | Typical device | Alpine cross-toolchain pkg | Rust target triple |
|---|---|---|---|
| `aarch64` | MT7986, ipq807x, newer Qualcomm | `gcc-aarch64-none-elf` *(or native)* | `aarch64-unknown-linux-musl` |
| `armv7l` | ipq40xx, older Qualcomm, RPi | `gcc-armv7-none-eabi` | `armv7-unknown-linux-musleabihf` |
| `mipsel` | MT7621 (Xiaomi, Netgear, Linksys) | `gcc-mipsel-linux-musl` | `mipsel-unknown-linux-musl` |
| `mips` | ath79 (older TP-Link, Ubiquiti) | `gcc-mips-linux-musl` | `mips-unknown-linux-musl` |
| `x86_64` | OpenWrt on a PC or VM | *(native)* | `x86_64-unknown-linux-musl` |

The simplest case is when your **Alpine host arch matches your router arch**
(e.g. both are aarch64). If they match, skip the cross-compilation section
entirely and just build natively.

---

## Step 1 — Prepare Alpine

Docker Alpine:

```bash
docker run --name alpine-builder --rm -it -v "$(pwd)":/src -w /src alpine:3.23 sh
```

Update package lists and install the core build dependencies:

NOTE: skip this section and go to rustup if u want build for arm7 arch!

```sh
echo "https://mirror.arvancloud.ir/alpine/v3.23/main" > /etc/apk/repositories && \
echo "https://mirror.arvancloud.ir/alpine/v3.23/community" >> /etc/apk/repositories

apk update
apk add --no-cache \
    rust \
    cargo \
    git \
    musl-dev \
    ca-certificates
```

`musl-dev` provides the musl headers and static library. `rust` and `cargo`
from Alpine are compiled against musl themselves, so every crate they compile
links against musl by default.

Confirm the installed Rust version:

```sh
rustc --version
cargo --version
```

Alpine ships a recent stable Rust. If the project requires a newer version than
Alpine provides, install `rustup` instead (see the note at the bottom of this
guide), but in most cases Alpine's packaged Rust is fine.

---

## Step 2 — Clone the Repository

```sh
git clone https://github.com/therealaleph/sni-spoofing-rust.git
cd sni-spoofing-rust
```

---

## Step 3A — Native Build (host arch = router arch)

If your Alpine machine has the same CPU architecture as your router, just build:

```sh
cargo build --release --bin sni-spoof-rs
```

The binary lands at:

```
target/release/sni-spoof-rs
```

Verify it is statically linked:

```sh
file target/release/sni-spoof-rs
# Expected: ELF 64-bit LSB executable, ... statically linked
```

Skip to **Step 4** to copy it to the router.

---

## Step 3B — Cross-Compilation (host arch ≠ router arch)

This section covers building on an **x86-64 Alpine host** for a different
router architecture. The steps are the same for any combination — only the
package name and target triple change.

### Install the cross-compiler for your target

Alpine's community repository ships gcc cross-toolchains for common embedded
targets. Enable community if you have not already:

```sh
# Check your current repositories
cat /etc/apk/repositories

# Add community if it is not listed (adjust version to match your Alpine)
echo "https://dl-cdn.alpinelinux.org/alpine/v$(cat /etc/alpine-release | cut -d. -f1,2)/community" \
    >> /etc/apk/repositories
apk update
```

Install the cross-compiler for your router's architecture:

```sh
# aarch64 (ARM64)
apk add --no-cache gcc-aarch64-none-elf

# ARMv7 hard-float
apk add --no-cache gcc-arm-none-eabi

# MIPS little-endian (MT7621)
apk add --no-cache gcc-mipsel-linux-musl 

# MIPS big-endian (ath79)
apk add --no-cache gcc-mips-linux-musl
```

> **Note:** Alpine's cross package names may vary slightly between releases.
> If a package is not found, search for it:
>
> ```sh
> apk search gcc-mipsel
> apk search musl-cross
> ```

### Find the cross-compiler binary name

After installation, confirm the exact binary name:

```sh
# Example for mipsel:
ls /usr/bin/ | grep mipsel
# Should show: mipsel-linux-musl-gcc  (or similar)
```

### Add the Rust target

```sh
# You need rustup for this step if using Alpine's packaged rust.
# Alpine's cargo can add targets via rustup if rustup is also installed,
# or you can install rust via rustup directly (see note at end of guide).
# The cleanest way on Alpine is to install rustup alongside the system rust:

apk add --no-cache rustup
rustup-init -y --no-modify-path --profile minimal
source "$HOME/.cargo/env"

# Now add your target — pick one:
rustup target add aarch64-unknown-linux-musl
rustup target add armv7-unknown-linux-musleabihf
rustup target add mipsel-unknown-linux-musl
rustup target add mips-unknown-linux-musl
```

### Configure Cargo's linker

Create or edit `~/.cargo/config.toml` and add the linker for your target.
Use the exact binary name you found in `/usr/bin/` above:

```toml
# MIPS little-endian (MT7621)
[target.mipsel-unknown-linux-musl]
linker = "mipsel-linux-musl-gcc"

# MIPS big-endian (ath79)
[target.mips-unknown-linux-musl]
linker = "mips-linux-musl-gcc"

# aarch64
[target.aarch64-unknown-linux-musl]
linker = "aarch64-none-elf-gcc"

# ARMv7 hard-float
[target.armv7-unknown-linux-musleabihf]
linker = "armv7-none-eabi-gcc"
```

You only need the block that matches your target — the others can be omitted.

ARMv7 musl linker :

```bash
apk add --no-cache curl tar

# Download the real Linux musl cross-compiler
curl -fsSL https://musl.cc/armv7l-linux-musleabihf-cross.tgz -o /tmp/tc.tgz
tar xf /tmp/tc.tgz -C /opt
rm /tmp/tc.tgz

# Verify
/opt/armv7l-linux-musleabihf-cross/bin/armv7l-linux-musleabihf-gcc --version
```

`~/.cargo/config.toml`

```
[target.armv7-unknown-linux-musleabihf]
linker = "/opt/armv7l-linux-musleabihf-cross/bin/armv7l-linux-musleabihf-gcc"
```

### Build

```sh
# MIPS little-endian (MT7621)
cargo build --release --bin sni-spoof-rs --target mipsel-unknown-linux-musl

# MIPS big-endian (ath79)
cargo build --release --bin sni-spoof-rs --target mips-unknown-linux-musl

# aarch64
cargo build --release --bin sni-spoof-rs --target aarch64-unknown-linux-musl

# ARMv7
cargo build --release --bin sni-spoof-rs --target armv7-unknown-linux-musleabihf
```

The binary lands at:

```
target/<your-target>/release/sni-spoof-rs
```

Verify:

```sh
file target/mipsel-unknown-linux-musl/release/sni-spoof-rs
# Expected: ELF 32-bit LSB executable, MIPS, ... statically linked
```

---

## Step 4 — Copy the Binary to the Router

From the Alpine build machine:

```sh
# Native build:
scp target/release/sni-spoof-rs root@192.168.1.1:/usr/local/bin/sni-spoof-rs

# Cross build (adjust target triple):
scp target/mipsel-unknown-linux-musl/release/sni-spoof-rs \
    root@192.168.1.1:/usr/local/bin/sni-spoof-rs

ssh root@192.168.1.1 "chmod +x /usr/local/bin/sni-spoof-rs"
```

> If `/usr/local/bin` does not exist on your router, use `/usr/bin/` or create
> it first with `mkdir -p /usr/local/bin`.

---

## Step 5 — Create the Config File on the Router

Resolve your VPN server's domain to get its Cloudflare IP (run this from the
Alpine build machine or any outside machine):

```sh
nslookup your-server-domain.com
# Cloudflare IPs are typically in:
# 104.x, 172.64–172.71.x, 188.114.x, 162.159.x, 141.101.x
```

SSH into the router and create the config:

```sh
ssh root@192.168.1.1

mkdir -p /etc/sni-spoof-rs

cat > /etc/sni-spoof-rs/config.json << 'EOF'
{
  "graceful_shutdown_sec": 0,
  "listeners": [
    {
      "listen": "127.0.0.1:40443",
      "connect": "CLOUDFLARE_IP:443",
      "fake_sni": "security.vercel.com",
      "conn_timeout_sec": 5,
      "handshake_timeout_sec": 2,
      "keepalive_time_sec": 11,
      "keepalive_interval_sec": 2
    }
  ]
}
EOF
```

Replace `CLOUDFLARE_IP` with the resolved IP.

To share the forwarder with all devices on the LAN, change `127.0.0.1:40443`
to `0.0.0.0:40443`.

### Finding a working `fake_sni`

If `security.vercel.com` does not bypass DPI on your ISP, run the built-in
scanner from the Alpine build machine (which is on the same network):

```sh
# On the Alpine machine, not the router:
sudo ./target/release/sni-spoof-rs scan
sudo ./target/release/sni-spoof-rs scan -o working.txt
```

Pick any `ok` result and use it as `fake_sni`.

---

## Step 6 — Test Manually on the Router

```sh
sni-spoof-rs /etc/sni-spoof-rs/config.json
```

With verbose logging:

```sh
RUST_LOG=info sni-spoof-rs /etc/sni-spoof-rs/config.json
```

Expected log output when working correctly:

- `fake ClientHello injected`
- `server ACK confirmed, fake was ignored`

Press `Ctrl-C` to stop after testing.

---

## Step 7 — Install as a Persistent Service

Create `/etc/init.d/sni-spoof-rs` on the router:

```sh
cat > /etc/init.d/sni-spoof-rs << 'EOF'
#!/bin/sh /etc/rc.common

START=99
STOP=10
USE_PROCD=1

start_service() {
    procd_open_instance
    procd_set_param command /usr/local/bin/sni-spoof-rs /etc/sni-spoof-rs/config.json
    procd_set_param env RUST_LOG=warn
    procd_set_param respawn 3600 5 0
    procd_set_param stdout 1
    procd_set_param stderr 1
    procd_close_instance
}
EOF

chmod +x /etc/init.d/sni-spoof-rs
/etc/init.d/sni-spoof-rs enable
/etc/init.d/sni-spoof-rs start
```

Check status and logs:

```sh
/etc/init.d/sni-spoof-rs status
logread | grep sni-spoof
```

---

## Step 8 — Integrate with Xray on the Router

`sni-spoof-rs` is a TCP forwarder only — you still need Xray for the VLESS /
VMess / Trojan protocol layer. Download the correct Xray binary for your router
arch from [XTLS/Xray-core releases](https://github.com/XTLS/Xray-core/releases).

In your Xray `config.json`, change only the outbound `address` and `port` to
point at the local `sni-spoof-rs` listener. All other fields — UUID, SNI, host,
path, transport, fingerprint — stay exactly as they were:

```json
{
  "outbounds": [
    {
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "127.0.0.1",
            "port": 40443,
            "users": [{ "id": "YOUR-UUID", "encryption": "none" }]
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "tlsSettings": {
          "serverName": "your-real-sni.example.com",
          "fingerprint": "chrome"
        },
        "wsSettings": {
          "path": "/your-path",
          "headers": { "Host": "your-real-sni.example.com" }
        }
      }
    }
  ]
}
```

---

## Troubleshooting

**`Exec format error` when running the binary on the router**
Wrong architecture. Re-check `uname -m` on the router and rebuild for the
correct target.

**`dynamically linked` shown by `file`**
The musl cross-linker was not used. On Alpine native builds this should not
happen — confirm `musl-dev` is installed. For cross builds, double-check the
linker name in `~/.cargo/config.toml` matches what is actually in `/usr/bin/`.

**`error: linker 'mipsel-linux-musl-gcc' not found`**
The cross package did not install the expected binary name. Run
`ls /usr/bin | grep mipsel` to find the actual name and update
`~/.cargo/config.toml`.

**`timeout waiting for fake ACK` on the router**

- The Cloudflare IP in `connect` may be stale — re-resolve the domain.
- Try a different `fake_sni` using the scanner.
- Run `ip route` on the router to confirm the right WAN interface is used.

**Not enough flash storage on the router**
Run `df -h` on the router. The binary is roughly 2–3 MB. If the overlay
partition is full, place the binary on a USB drive or configure extroot.

**Alpine's packaged Rust is too old for the project**
Install via rustup instead of apk:

```sh
apk add --no-cache curl
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
```

Then continue from Step 2. The rest of the guide is identical.

---

## Quick Reference

```sh
# On Alpine build machine — native build (same arch as router):
apk add rust cargo git musl-dev ca-certificates
git clone https://github.com/therealaleph/sni-spoofing-rust.git
cd sni-spoofing-rust
cargo build --release --bin sni-spoof-rs
scp target/release/sni-spoof-rs root@192.168.1.1:/usr/local/bin/
ssh root@192.168.1.1 "chmod +x /usr/local/bin/sni-spoof-rs"

# On the router:
# Config:  /etc/sni-spoof-rs/config.json
# Test:    sni-spoof-rs /etc/sni-spoof-rs/config.json
# Service: /etc/init.d/sni-spoof-rs enable && /etc/init.d/sni-spoof-rs start
# Logs:    logread | grep sni-spoof
```
