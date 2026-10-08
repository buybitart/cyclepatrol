# Attack surface and external dependencies

CyclePatrol is a **single-process CLI tool**, run as root on Linux / Android. It has no HTTP server,
no login, no database, and no network listener. So the attack surface is not web routes. It is:

- radio input from the air (untrusted),
- operator-controlled input (CLI flags, env vars, files),
- output from external tools (which comes from the air),
- the local filesystem it writes to.

This page maps those inputs. It is for system understanding and defensive review. It has no
exploitation steps.

## Entry points (where data enters)

```mermaid
flowchart TB
    subgraph UNTRUSTED[Untrusted input]
        RF[RF air: beacons, probe resp,<br/>EAPOL, PMKID, 4-way handshake]
    end
    subgraph OPER[Operator-controlled input]
        CLI[CLI flags: --recon, --baseline-*, ...]
        ENV[env vars: IFACE, allowlists,<br/>wordlists, REVEAL_CREDS, timeouts]
        FILES[files: ACTIVE_ALLOWLIST_FILE,<br/>wordlists, WPA_AUDIT_EXTERNAL_TOOL script]
    end
    subgraph PROC[cp.sh process - root]
        PARSE[parse_scan awk<br/>+ normalize_bssid + validate_env]
        CORE[scan / enrich / orchestrate]
    end
    subgraph TOOLOUT[External tool output - from RF]
        TOUT[reaver / pixiewps / wpa_supplicant logs,<br/>hcx 22000, tcpdump pcap, airodump csv]
    end
    OUT[(LOG_DIR files)]

    RF --> PARSE
    TOUT --> CORE
    CLI --> CORE
    ENV --> PARSE
    FILES --> CORE
    PARSE --> CORE
    CORE --> OUT
```

| Entry point | Reaches code at | Validation / handling |
|---|---|---|
| `iw scan` beacon text | `parse_scan` | awk parser; values escaped by `json_escape` / `_html_escape` |
| BSSID from scan | `process_target` | `normalize_bssid` + `check_bssid` before shell use |
| CLI flags | `main` | fixed `case` list; unknown flag → warning |
| env vars (numeric) | `validate_env` | range-checked; bad value → default or `FATAL` |
| `IFACE` | `validate_env` | regex `^[a-zA-Z0-9._-]{1,32}$` |
| `LOG_DIR` | `validate_env` | rejects `..` and leading `-` |
| `USER_PROVIDED_PIN` | `validate_env` | must be 8 digits |
| allowlist file / CSV | `allowlist_load_file` / `_csv` | each token normalized as BSSID |
| capture files (pcap, 22000) | capture modules | parsed by `tcpdump` / `hcxpcapngtool` / `aircrack`/`hashcat` |
| `WPA_AUDIT_EXTERNAL_TOOL` | `wpa_audit_external_dispatch` | executed if `WPA_AUDIT_MODE=external`; see security.md O4 |

## External dependencies and direction

```mermaid
flowchart LR
    CP[cp.sh]

    subgraph RFTOOLS[RF tools - act on the air]
        RE[reaver / bully]
        PX[pixiewps]
        WS[wpa_supplicant / wpa_cli]
        HCX[hcxdumptool]
        TD[tcpdump]
        AD[airodump-ng]
        AR[aireplay-ng / mdk4]
        ET[eaphammer / hostapd-wpe]
    end

    subgraph OFFLINE[offline / helper tools]
        HC[hashcat / aircrack-ng]
        HXT[hcxpcapngtool]
        TS[tshark]
        AG[airgraph-ng]
        JQ[jq]
    end

    subgraph LOCAL[local services]
        GPSD[gpsd 127.0.0.1]
        TL[termux-location - Android bridge]
    end

    APT[apt-get -> OS package repos]
    VKB[vendor/oui.json + vendors.json]
    IW[iw / ip / nmcli / macchanger -> Wi-Fi driver]

    CP --> RE & PX & WS & HCX & TD & AD & AR & ET
    CP --> HC & HXT & TS & AG & JQ
    CP --> IW
    CP --> GPSD
    CP --> TL
    CP --> VKB
    CP -->|install missing tools| APT
    RE & PX & WS & HCX & TD & AD & AR & ET --> AIR((RF air))
```

Notes:
- The **only non-radio network egress** is `apt-get` in `auto_install_deps`. The code
  comment and logic restrict it to the OS package manager (signed repos), no `curl | bash`.
- `gpsd` is reached on `127.0.0.1`; `termux-location` is a local Android bridge. No remote server.
- `vendor/oui.json` and `vendor/vendors.json` are read locally (`ENABLE_VENDOR_KB`).

## Runtime / deployment

No Docker, Compose, Kubernetes, CI, or cloud config exists in the repo. "Deployment" is: copy the
script to a Linux / Kali NetHunter host and run it as root with a monitor-capable Wi-Fi adapter.

```mermaid
flowchart LR
    OP[Operator host<br/>Kali / Android NetHunter<br/>root shell] --> SH[cp.sh]
    SH --> NIC[Wi-Fi adapter<br/>IFACE default wlan2<br/>monitor mode]
    NIC <--> AIR((RF airspace:<br/>APs + clients))
    SH --> LOG[(LOG_DIR<br/>default /sdcard/Download/wifi)]
    SH -. optional .-> GPS[GPS: gpsd / termux-location]
```

## What does NOT apply (verified absent)

The spec lists web-app concepts. This tool has none of them. Confirmed from the repo:

| Concept | Status | Why |
|---|---|---|
| HTTP API / routes | none | no web server or router in code |
| Login / sessions / cookies | none | no interactive auth; root is required by the OS |
| JWT / tokens | none | not used |
| Database / ORM / migrations | none | state is flat files (`sar.jsonl`, `*.txt`, proof files) |
| Webhooks | none | no inbound callbacks |
| Queue / message broker / worker service | none | target loop is sequential in `run_scan_loop`; only per-tool child processes with `wait` + `timeout` |
| Scheduled jobs / cron | none | the loop sleeps `COOLDOWN` between rounds; no scheduler |
| Reverse proxy / TLS config | none | no network service to terminate |
| Docker / CI / CD | none | only `cp.sh`, `vendor/`, `README.md`, `LICENSE` in repo |

> `ENABLE_MULTITHREADING` / `MAX_CORES` only affect the `pixiewps` step inside
> `attack_pixie`. They do not create a parallel target pipeline.

The authorization / scope model and the confirmed security observations are in
[`security.md`](security.md). Trust boundaries are in [`architecture.md`](architecture.md#trust-boundaries).
