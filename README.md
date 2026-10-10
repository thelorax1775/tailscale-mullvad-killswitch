# tailscale-mullvad-killswitch

A small, fail-closed setup for routing a Linux box through a [Mullvad](https://mullvad.net) exit node over [Tailscale](https://tailscale.com), with automatic node rotation, a self-healing watchdog, and an nftables kill-switch that prevents your real IP from ever leaking.

If the tunnel ever drops or can't be verified, **all non-tunnel egress is blocked** — the system fails *closed*, not open.

## How it works

There are three moving parts:

1. **Kill-switch (`nftables/mullvad-killswitch.nft`)** — an `output`-hook firewall with a default `drop` policy. It only permits traffic out the Tailscale interface (`tailscale0`), Tailscale's own encrypted underlay (marked `0x80000`), loopback, established flows, and your LAN. Anything else — raw internet egress over the physical NIC — is dropped. This is the leak protection, and it's enforced independently of the script.

2. **Rotation / self-heal script (`mullvad-rotate.sh`)** — picks a random *online* Mullvad exit node, switches to it, and then **verifies** that traffic actually flows before considering it good. If verification fails it transparently tries another node until one works or candidates are exhausted. Because the kill-switch blocks raw egress, a successful outbound connection is proof the tunnel is live.

3. **systemd units (`systemd/`)** — load the kill-switch at boot (before networking comes up) and run the script on a timer as a watchdog.

## Requirements

- Linux with `systemd` and `nftables`
- [Tailscale](https://tailscale.com/download) installed and logged in
- A Mullvad exit node enabled on your tailnet (Tailscale's [Mullvad add-on](https://tailscale.com/kb/1258/mullvad-exit-nodes))
- `jq` (optional, used to detect the current node so rotation prefers a *different* one)

## Installation

Run as root (or with `sudo`).

### 1. Kill-switch

```bash
sudo mkdir -p /etc/nftables.d
sudo cp nftables/mullvad-killswitch.nft /etc/nftables.d/mullvad-killswitch.nft
```

**Important:** edit the file and set `LAN` to match your local subnet if it isn't `192.168.1.0/24`:

```bash
sudo sed -i 's#192.168.1.0/24#YOUR.SUBNET.0/24#' /etc/nftables.d/mullvad-killswitch.nft
```

Apply it immediately to test (idempotent — safe to re-run):

```bash
sudo /usr/sbin/nft -f /etc/nftables.d/mullvad-killswitch.nft
```

### 2. Rotation script

```bash
sudo cp mullvad-rotate.sh /usr/local/bin/mullvad-rotate.sh
sudo chmod +x /usr/local/bin/mullvad-rotate.sh
```

### 3. systemd units

```bash
sudo cp systemd/*.service systemd/*.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now mullvad-killswitch.service
sudo systemctl enable --now mullvad-healthcheck.timer
```

The kill-switch service runs once at boot and stays active. The healthcheck timer fires 2 minutes after boot and every 15 minutes thereafter, re-acquiring a node only if connectivity is down.

## Usage

Rotate to a fresh, verified node right now:

```bash
sudo /usr/local/bin/mullvad-rotate.sh
```

Watchdog mode — does nothing if the tunnel is healthy, only re-acquires when connectivity is down (this is what the timer runs):

```bash
sudo /usr/local/bin/mullvad-rotate.sh --check-only
```

Watch the log:

```bash
tail -f /var/log/mullvad-rotate.log
```

Check timer status / next run:

```bash
systemctl status mullvad-healthcheck.timer
systemctl list-timers mullvad-healthcheck.timer
```

## Configuration

Edit the variables at the top of `mullvad-rotate.sh`:

| Variable | Default | Description |
|---|---|---|
| `COUNTRY_FILTER` | `""` | Restrict to one country, e.g. `"USA"`. Empty = any country. |
| `ALLOW_LAN` | `true` | Keep LAN access (web UI / SSH) while the exit node is active. |
| `LOG_FILE` | `/var/log/mullvad-rotate.log` | Where run output is logged. |
| `CHECK_TIMEOUT` | `8` | Seconds per connectivity probe. |
| `TUNNEL_SETTLE` | `3` | Seconds to wait after switching before probing. |

To change the watchdog frequency, edit `OnUnitActiveSec` in `systemd/mullvad-healthcheck.timer` and run `sudo systemctl daemon-reload`.

## Verifying it fails closed

With the kill-switch loaded, clear the exit node and confirm egress is blocked:

```bash
sudo tailscale set --exit-node=
curl -m 5 https://1.1.1.1   # should time out / fail
```

Then re-run the script and confirm connectivity returns:

```bash
sudo /usr/local/bin/mullvad-rotate.sh
curl -m 5 https://api.ipify.org && echo   # should print a Mullvad IP
```

## Uninstall

```bash
sudo systemctl disable --now mullvad-healthcheck.timer mullvad-killswitch.service
sudo rm /etc/systemd/system/mullvad-healthcheck.{service,timer} \
        /etc/systemd/system/mullvad-killswitch.service \
        /usr/local/bin/mullvad-rotate.sh \
        /etc/nftables.d/mullvad-killswitch.nft
sudo systemctl daemon-reload
sudo nft delete table inet mullvad_killswitch
```

## Project status and related repos

Stable, single-host setup. No changes since the initial release (June 2026).

To roll this out to many Proxmox guests at once, see
[tailscale-mullvad-bulk-deploy](https://github.com/thelorax1775/tailscale-mullvad-bulk-deploy).
It vendors these files in its `assets/` directory, and its copies have since
gained features this repo does not have: a `LAN_SUBNETS` bypass in
`mullvad-rotate.sh`, so other LAN subnets stay reachable while an exit node is
active, and templated `LAN`/country settings. If you want those features, use
the files from that repo.

## License

MIT — see [LICENSE](LICENSE).
