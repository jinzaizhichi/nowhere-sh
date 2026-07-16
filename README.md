# nowhere-sh

[简体中文](README.zh-CN.md)

A one-click Linux VPS deployment script for Nowhere Portal. It supports both
the legacy Anywhere-compatible protocol and the new Native Vector protocol
introduced in Nowhere v1.5.

The script installs an explicit release version, creates a systemd service,
and prints the correct client URL for the selected protocol generation.

## Features

- Detects `x86_64` / `aarch64` and `gnu` / `musl` Linux packages automatically.
- Installs Nowhere to `/usr/local/bin/nowhere`.
- Stores configuration in `/etc/nowhere/nowhere.env` with `0600` permissions.
- Creates and manages `/etc/systemd/system/nowhere.service`.
- Supports `mix`, `tcp`, and `udp` listener modes.
- Supports `tls=1` ephemeral self-signed certificates and `tls=2` PEM certificates.
- Supports rate limits, outbound source address, log level, and Anywhere TCP pool.
- Supports Nowhere `v1.2.4+` SOCKS5 outbound upstream via `socks`.
- Lists the 10 most recent GitHub Releases for numeric version selection.
- Supports Anywhere links for v1.4 and earlier, including version-aware `net=`
  and `up=` / `down=` parameters.
- Supports v1.5+ Native Vector URLs with `sni`, local SOCKS5 ingress, split
  carriers, and a TCP pool up to 256.

## Protocol Compatibility

Nowhere v1.5 uses a new wire protocol and is not compatible with the current
Anywhere client. Choose the mode that matches your client:

| Mode | Portal version | Client | Client URL |
| --- | --- | --- | --- |
| Anywhere compatible | v1.4.0 and earlier | Anywhere | `nowhere://...` |
| Native Vector | v1.5.0 and later | Same-version Nowhere binary | `vector://...` |

The default Anywhere entry installs v1.4.0. The default Native Vector entry
installs v1.5.0. Portal and Native Vector must use compatible protocol versions.

## Requirements

- A Linux VPS with systemd.
- `curl` and `tar`.
- Debian, Ubuntu, Rocky Linux, AlmaLinux, CentOS Stream, and similar distributions are recommended.

## Quick Start

Download the script first:

```bash
curl -fsSL https://raw.githubusercontent.com/chikacya/nowhere-sh/main/nowhere-vps.sh -o nowhere-vps.sh
chmod +x nowhere-vps.sh
sudo bash nowhere-vps.sh
```

Running the script without arguments opens the interactive menu:

```text
1) Install/Reinstall Anywhere-compatible v1.4.0
2) Install/Reinstall Native Vector v1.5.0
3) Quick default Anywhere-compatible install
4) Reconfigure the current protocol mode
5) Select and install one of the 10 most recent Releases
6) Start service
7) Stop service
8) Restart service
9) Show status
10) Show logs
11) Print client URLs/commands
12) Show tls=1 self-signed certificate SHA-256
13) Uninstall service
0) Exit
```

Choose `1` for Anywhere or `2` for Native Vector, then press Enter through the
wizard to accept the defaults.

One-line install is also supported:

```bash
curl -fsSL https://raw.githubusercontent.com/chikacya/nowhere-sh/main/nowhere-vps.sh | sudo bash -s -- install-legacy
```

Use `install-vector` instead to install the v1.5 Native Vector mode.

## Release Selection

Choose menu item `5`, or run:

```bash
sudo bash nowhere-vps.sh versions
```

The script queries GitHub and shows the 10 most recent releases. Enter a number
to install that exact version. Versions before v1.5 automatically use the
Anywhere-compatible wizard; v1.5 and later automatically use the Native Vector
wizard. The selected version is saved in `/etc/nowhere/nowhere.env`.

## Recommended Deployment

For daily use, use a domain name and a valid TLS certificate:

```bash
sudo NOWHERE_PUBLIC_HOST=proxy.example.com \
  NOWHERE_PORT=443 \
  NOWHERE_NET=mix \
  NOWHERE_TLS=2 \
  NOWHERE_CRT=/etc/letsencrypt/live/proxy.example.com/fullchain.pem \
  NOWHERE_TLS_KEY=/etc/letsencrypt/live/proxy.example.com/privkey.pem \
  bash nowhere-vps.sh install --yes
```

For quick testing, you can use the default `tls=1` mode:

```bash
sudo bash nowhere-vps.sh install --yes
```

Note: `tls=1` creates an ephemeral self-signed certificate when Nowhere starts.
The certificate changes after every restart. Use `tls=2` for long-lived deployments.

## Self-Signed Certificate SHA-256

The default `tls=1` mode creates an in-memory self-signed certificate. The script
automatically tries to print the current certificate SHA-256 fingerprint after
install, restart, or update:

```text
Current tls=1 self-signed certificate SHA-256 fingerprint:
  AA:BB:CC:...
```

You can also print it manually:

```bash
sudo bash nowhere-vps.sh fingerprint
```

Or choose menu item `12`.

The script first reads the `CERT_SHA256|...` log field introduced in Nowhere
`v1.2.5+`. If that is not available, it falls back to local TLS probing or older
log matching.

Because `tls=1` certificates live in memory, the fingerprint changes after every
Nowhere restart. Production deployments should use `tls=2` with a stable
certificate.

## Installer Wizard

Choosing menu item `1` or `2` starts the corresponding interactive wizard.
Every prompt has a default value:

```text
Public domain/IP for client connections [1.2.3.4]:
Listen address, empty means IPv4/IPv6 wildcard:
Listen port [2077]:
Shared Key [random value]:
Spec Seed [random value]:              # Anywhere mode only
Listener mode mix/tcp/udp [mix]:
Vector local SOCKS5 listener [127.0.0.1:1080]:  # Vector mode only
```

If you do not want to customize anything, press Enter through all prompts. The
script shows a configuration summary before applying it; press Enter again to
confirm.

## SOCKS5 Upstream

If you want all outbound target traffic from Nowhere Portal to go through a
SOCKS5 proxy, configure `NOWHERE_SOCKS`:

```bash
sudo NOWHERE_PUBLIC_HOST=proxy.example.com \
  NOWHERE_PORT=443 \
  NOWHERE_NET=mix \
  NOWHERE_TLS=2 \
  NOWHERE_CRT=/etc/letsencrypt/live/proxy.example.com/fullchain.pem \
  NOWHERE_TLS_KEY=/etc/letsencrypt/live/proxy.example.com/privkey.pem \
  NOWHERE_SOCKS=user:pass@127.0.0.1:1080 \
  bash nowhere-vps.sh install --yes
```

Supported `NOWHERE_SOCKS` formats:

```text
none
host:port
user:pass@host:port
[2001:db8::10]:1080
user:pass@[2001:db8::10]:1080
```

`NOWHERE_SOCKS` is a server-side outbound setting. It is written into the Portal
URL, but it is not added to the `nowhere://` link imported by Anywhere. From the
client perspective, Anywhere still connects only to your Nowhere Portal.

## Management Commands

```bash
sudo bash nowhere-vps.sh configure
sudo bash nowhere-vps.sh versions
sudo bash nowhere-vps.sh start
sudo bash nowhere-vps.sh stop
sudo bash nowhere-vps.sh restart
sudo bash nowhere-vps.sh status
sudo bash nowhere-vps.sh logs
sudo bash nowhere-vps.sh link
sudo bash nowhere-vps.sh fingerprint
sudo bash nowhere-vps.sh uninstall
```

Command notes:

- `configure`: edit the current protocol mode and restart the service if enabled.
- `versions`: list the 10 most recent releases and safely switch version and mode.
- `update --version vX.Y.Z`: install one explicit release and use its matching mode.
- `logs`: follow systemd logs.
- `link`: print Anywhere links or Native Vector URLs again.
- `fingerprint`: print the current `tls=1` self-signed certificate SHA-256 fingerprint.
- `uninstall`: remove the binary and systemd service, but keep `/etc/nowhere` so keys are not deleted accidentally.

## Parameters

All options can be entered in the wizard, set as environment variables, or passed
as command-line flags.

| Environment variable | CLI flag | Default | Description |
| --- | --- | --- | --- |
| `NOWHERE_PROTOCOL` | `--protocol` | `legacy` | `legacy` for Anywhere or `vector` for v1.5+ Native Vector |
| `NOWHERE_VERSION` | `--version` | mode-dependent | Exact release tag, such as `v1.4.0` or `v1.5.0` |
| `NOWHERE_PUBLIC_HOST` | `--public-host` | auto-detected | Domain or public IP used in generated client URLs |
| `NOWHERE_LISTEN_HOST` | `--listen-host` | empty | Listen address; empty means IPv4/IPv6 wildcard |
| `NOWHERE_PORT` | `--port` | `2077` | Portal listen port |
| `NOWHERE_KEY` | `--key` | random | Nowhere shared key |
| `NOWHERE_SPEC` | `--spec` | random | Legacy protocol seed; removed and rejected in v1.5+ mode |
| `NOWHERE_NET` | `--net` | `mix` | `mix`, `tcp`, or `udp` |
| `NOWHERE_TLS` | `--tls` | `1` | `1` ephemeral self-signed cert, `2` PEM cert |
| `NOWHERE_CRT` | `--crt` | empty | Certificate chain path for `tls=2` |
| `NOWHERE_TLS_KEY` | `--tls-key` | empty | Private key path for `tls=2` |
| `NOWHERE_ALPN` | `--alpn` | `now/1` | TLS/QUIC ALPN |
| `NOWHERE_RATE` | `--rate` | `0` | Client-to-target Mbps limit; `0` disables it |
| `NOWHERE_ETAR` | `--etar` | `0` | Target-to-client Mbps limit; `0` disables it |
| `NOWHERE_DIAL` | `--dial` | `auto` | Outbound source IP or `auto` |
| `NOWHERE_SOCKS` | `--socks` | `none` | SOCKS5 outbound upstream |
| `NOWHERE_LOG` | `--log` | `info` | `none`, `debug`, `info`, `warn`, `error`, or `event` |
| `NOWHERE_POOL` | `--pool` | `5` | TCP pool: Anywhere `0..9`, Native Vector `0..256` |
| `NOWHERE_VECTOR_SOCKS` | `--vector-socks` | `127.0.0.1:1080` | Native Vector local SOCKS5 ingress listener |
| `NOWHERE_VECTOR_SNI` | `--sni` | `none` or TLS domain | Native Vector certificate verification name |

## Firewall

If `NOWHERE_NET=mix`, open both TCP and UDP on the selected port:

```bash
sudo ufw allow 443/tcp
sudo ufw allow 443/udp
```

firewalld example:

```bash
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=443/udp
sudo firewall-cmd --reload
```

If `NOWHERE_NET=tcp`, open TCP only. If `NOWHERE_NET=udp`, open UDP only.

## Client URLs

### Anywhere Mode

After installation, the script prints:

- `nowhere://...`
- `anywhere://add-proxy?link=...`

For Nowhere v1.3-v1.4, the generated Anywhere links use `up=` and `down=`:

- `up=udp&down=udp`: QUIC/UDP, recommended when UDP is reachable.
- `up=tcp&down=tcp`: TLS/TCP fallback with the configured TCP pool.
- `up=tcp&down=udp` and `up=udp&down=tcp`: asymmetric carrier links, printed
  only for `NOWHERE_NET=mix` without a SOCKS5 upstream.

The old `net=` client import parameter is still treated as legacy by Anywhere,
and is generated automatically for v1.2.x. Releases v1.3-v1.4 use `up=` and
`down=` directly.

On iPhone, iPad, or Apple TV, copy a `nowhere://` link into Anywhere. If your
system recognizes Anywhere deep links, you can open the corresponding
`anywhere://add-proxy?link=...` link directly.

If you forgot to save the link, run this on the VPS:

```bash
sudo bash nowhere-vps.sh link
```

### Native Vector Mode

For v1.5+, the script prints one or more `vector://` URLs and a command such as:

```bash
nowhere 'vector://shared-key@relay.example:2077?up=udp&down=udp&sni=relay.example&socks=127.0.0.1%3A1080'
```

Run this on the client device with a matching v1.5+ Nowhere binary, then point
applications at the local SOCKS5 listener. `sni=none` disables certificate
verification and is the default for `tls=1`; use a CA-trusted certificate and
an explicit DNS SNI for production.

## File Locations

```text
/usr/local/bin/nowhere
/etc/nowhere/nowhere.env
/etc/systemd/system/nowhere.service
```

Check service status:

```bash
systemctl status nowhere
```

Show configuration:

```bash
sudo cat /etc/nowhere/nowhere.env
```

## Troubleshooting

Follow logs:

```bash
sudo bash nowhere-vps.sh logs
```

Check listening ports:

```bash
ss -lntup | grep nowhere
```

Restart the service:

```bash
sudo bash nowhere-vps.sh restart
```

Common issues:

- Cannot connect: check VPS security group, firewall, and TCP/UDP port rules for `NOWHERE_NET`.
- Certificate errors: use `tls=2` in production and make sure `NOWHERE_PUBLIC_HOST` matches the certificate name.
- QUIC does not work: many cloud firewalls allow TCP only by default; open UDP separately.
- SOCKS5 does not work: make sure the proxy in `NOWHERE_SOCKS` is reachable from the VPS and credentials are correct.

## Upstream Projects

- Nowhere: https://github.com/NodePassProject/Nowhere
- Anywhere: https://github.com/NodePassProject/Anywhere
