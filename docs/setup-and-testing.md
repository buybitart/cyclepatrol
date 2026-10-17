# Setup and testing

## Requirements

| Need | Detail | Checked in |
|---|---|---|
| OS | Linux or Android (Kali NetHunter) | — |
| Shell | bash 5+ | script header |
| Privilege | root | `run_precheck` |
| Wi-Fi adapter | monitor mode + injection | `ensure_mode`, modules |
| Required tools | `ip iw wpa_supplicant wpa_cli pixiewps reaver awk sort grep sed tr timeout flock mktemp sha256sum` | `run_precheck` |
| Optional tools | `macchanger bully nmcli airodump-ng aireplay-ng hcxdumptool hcxpcapngtool tcpdump hostapd-wpe hashcat aircrack-ng tshark airgraph-ng gpsd jq eaphammer mdk4` | `run_precheck` |

If a required tool is missing, `run_precheck` stops with `FATAL`. If an optional tool is missing, the
matching module skips cleanly.

## Install

```bash
git clone https://github.com/buybitart/cyclepatrol
cd cyclepatrol
chmod +x cp.sh
```

On first run (the default) `auto_install_deps` installs missing tools with `apt-get` (Kali/Debian
only, root only, signed OS repos only, no `curl | bash`). To skip it:

```bash
sudo ./cp.sh --no-install        # or AUTO_INSTALL=0
```

## Run

```bash
sudo ./cp.sh                     # full loop (recon once, then scan loop)
sudo ./cp.sh --recon             # recon survey only, then exit
sudo IFACE=wlan1 ./cp.sh         # choose interface
sudo ./cp.sh --self-check        # passive posture self-check, writes proof + report, exits
```

See [`configuration.md`](configuration.md) for scope and module flags.

## Checks that need no root and no radio

### 1. Syntax check

```bash
bash -n cp.sh                    # parse only, reports syntax errors
```

### 2. Static analysis

```bash
shellcheck cp.sh                 # lint, install separately
```

The current build reports 0 shellcheck errors. The remaining warnings are reviewed and benign (local
variable names reused across functions, and one comma inside a single `airodump-ng` argument).

### 3. Function-level test harnesses

Development uses a set of Bash harnesses (`test_*.sh`) that `source cp.sh` and exercise single
functions and the report output without touching the radio. Each harness prints a
`RESULT pass=<n> fail=<n>` line and exits non-zero on any failure, so they run in a plain loop:

```bash
for t in test_*.sh; do bash "$t"; done
```

They cover the rule engine, logging, the adaptive triggers, the report model, the proof gate, the
posture evidence, and the Simple/Technical report split. They are not under version control in the
repo root yet. Keep them next to `cp.sh` to run them.

### 4. Built-in self-check

```bash
sudo ./cp.sh --self-check        # passive posture self-check, <=30s, writes proof/sci + reports/
```

### 5. GPS bridge test

```bash
sudo ./cp.sh --gps-test          # one location fix, then exit
```
