# Setup and testing

## Requirements

| Need | Detail | Checked in |
|---|---|---|
| OS | Linux or Android (Kali NetHunter) | — |
| Shell | bash 5+ | script header |
| Privilege | root | `run_precheck` |
| Wi-Fi adapter | monitor mode + injection | `ensure_mode`, modules |
| Required tools | `ip iw wpa_supplicant wpa_cli pixiewps reaver awk sort grep sed tr timeout flock mktemp sha256sum` | `run_precheck` |
| Optional tools | `macchanger bully nmcli airodump-ng aireplay-ng hcxdumptool hcxpcapngtool tcpdump hostapd-wpe hashcat aircrack-ng` + `tshark airgraph-ng gpsd jq eaphammer mdk4` | `run_precheck` |

If a required tool is missing, `run_precheck` stops with `FATAL`. If an optional tool is missing, the
matching module skips cleanly.

## Install

```bash
git clone https://github.com/buybitart/cyclepatrol
cd cyclepatrol
chmod +x cp.sh
```

Dependencies: on first run (default) `auto_install_deps` installs missing tools with `apt-get`
(Kali/Debian only, root only, signed OS repos only — no `curl | bash`). To skip:

```bash
sudo ./cp.sh --no-install        # or AUTO_INSTALL=0
```

## Run

```bash
sudo ./cp.sh                     # full loop (recon once, then scan loop)
sudo ./cp.sh --recon             # recon survey only, then exit
sudo IFACE=wlan1 ./cp.sh         # choose interface
```

See [`configuration.md`](configuration.md) for scope and module flags.

## Test / verify the script

There is **no automated test suite in the repo**. Use these checks:

### 1. Syntax check (no root, no radio)

```bash
bash -n cp.sh                    # parse only, reports syntax errors
```

### 2. Static analysis (if available)

```bash
shellcheck cp.sh                 # lint; not part of the repo, install separately
```

### 3. GPS bridge test

```bash
sudo ./cp.sh --gps-test          # one location fix, then exit
```

