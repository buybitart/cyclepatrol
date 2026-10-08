# CyclePatrol

![license](https://img.shields.io/badge/license-MIT-blue)
![platform](https://img.shields.io/badge/platform-Linux%20%7C%20Android%20(Kali%20NetHunter)-informational)
![shell](https://img.shields.io/badge/shell-bash%205%2B-green)
![version](https://img.shields.io/badge/version-1.3-lightgrey)

CyclePatrol is one Bash script, `cp.sh`. It is a **Wi-Fi security assessment tool** for the terminal.
It runs on Linux and Android (Kali NetHunter). It needs **root**.

The tool runs in a loop. In each round it:

1. scans nearby Wi-Fi networks,
2. parses each beacon and reads the security facts,
3. runs enabled assessment modules (WPS, PMKID, handshake, WPA2 dictionary, WPA3/SAE, DPP, enterprise EAP),
4. saves facts, evidence (proof files) and an HTML report.

> **Authorized use only.** CyclePatrol runs active wireless attacks.
> Use it only on networks you own or have written permission to test.

This documentation is written from the source code only.
It does not explain how to attack networks. It explains how the tool is built, how data moves, and where the security logic is.

---

## Facts

| Item | Value | Source |
|---|---|---|
| Tool name | `CyclePatrol` | `TOOL_NAME` |
| Version | `1.3` | `VERSION` |
| SAR / report version | `1` / `4` | `SAR_VERSION` / `REPORT_VERSION` |
| Language | Bash 5+ | — |
| Privilege | root required | `run_precheck` |
| Default interface | `wlan2` | `DEF_IFACE` |
| Default log dir | `/sdcard/Download/wifi` | `DEF_LOG_DIR` |
| Files in repo | `cp.sh`, `vendor/oui.json`, `vendor/vendors.json`, `README.md`, `LICENSE` | repo root |
| Tests / CI | none in repo | repo root |
| Config | environment variables + CLI flags only | `main` |

---

## Installation / Start

```bash
# 1. get the code
git clone https://github.com/buybitart/cyclepatrol   
cd cyclepatrol 

# 2. make it runnable
chmod +x cp.sh

# 3. start
./cp.sh
```

Help and the full option list:

```bash
./cp.sh --help
```

---

## Pipeline

```mermaid
flowchart TD
    M[main parse CLI] --> AI[auto_install_deps apt]
    AI --> PC[run_precheck validate_env root iface tmp dirs allowlists]
    PC --> RC{recon mode?}
    RC -- "--recon" --> RE[run_recon airodump-ng] --> RPT1[report_recon_*.html] --> END1[exit]
    RC -- default --> R0[run_recon once at start]
    R0 --> LOOP[run_scan_loop]
    LOOP --> MAC[stealth MAC rotate]
    MAC --> SCAN[scan_band 2.4 / 5 / 6E via iw scan]
    SCAN --> PARSE[parse_scan awk -> targets]
    PARSE --> PT[process_target per AP, up to MAX_TARGETS]
    PT --> REP[make_report report_*.html]
    REP --> SLEEP[sleep COOLDOWN] --> LOOP
```

See [`docs/architecture.md`](docs/architecture.md) and [`docs/data-flow.md`](docs/data-flow.md) for detail.

---

## Main components

| Component | Functions (examples) | Job |
|---|---|---|
| Startup / config | `main`, `validate_env`, `run_precheck`, `auto_install_deps` | parse flags, check root + interface + tools, build temp dirs |
| Interface control | `ensure_mode`, `iface_set_type`, `iface_set_channel`, `iface_do_scan` | monitor/managed mode, channel, `iw` scan |
| Stealth | `stealth_mac_rotate`, `stealth_gen_mac`, `stealth_wps_id_block` | random MAC, random WPS UUID |
| Scan + parse | `scan_band`, `parse_scan` | collect `iw scan` output, parse beacons into fields |
| Enrichment | `get_chip`, `get_cves`, `detect_dpp_vulns`, vendor KB lookups | OUI → vendor → chipset, CVE hints, DPP checks |
| Scope gate | `active_attack_permitted`, `allowlist_init_active` | decide if an active module may run on a BSSID |
| Assessment modules | `attack_pixie`, `attack_vendor`, `attack_pmkid`, `attack_deauth_handshake`, `attack_wpa_psk_brute`, `attack_wpa_enterprise_eviltwin` | the wireless tests (wrap external tools) |
| Offline crack | `offline_crack`, `_hashcat_22000`, `_aircrack_cap` | crack captured PMKID / handshake |
| Evidence + output | `sar_store_record`, `save_proof`, `make_report`, `recon_report` | JSONL facts, proof files, HTML reports |
| Location | `location_fix`, `location_nmea_feeder` | GPS tagging for war-driving (Android / gpsd) |

Full function map: [`docs/project-structure.md`](docs/project-structure.md).

---

## External tools used

CyclePatrol is an orchestrator. It calls standard Linux wireless tools as subprocesses.

| Tool | Used by | Purpose |
|---|---|---|
| `iw`, `ip`, `nmcli`, `macchanger` | interface + stealth | scan, mode, channel, MAC |
| `reaver` | `attack_pixie`, `attack_vendor`, `attack_null`, `attack_empty`, `attack_user_pin` | WPS registrar exchange |
| `pixiewps` | `attack_pixie` | WPS offline PIN recovery |
| `wpa_supplicant`, `wpa_cli`, `wpa_passphrase` | WPS, `verify_psk_live`, `attack_wpa_psk_brute` | association, live key check |
| `hcxdumptool`, `hcxpcapngtool` | `attack_pmkid`, `attack_pmkid_active` | PMKID capture → hashcat 22000 |
| `tcpdump` | `eapol_passive_capture` | passive EAPOL capture |
| `airodump-ng`, `aireplay-ng`, `mdk4` | `attack_deauth_handshake`, `run_recon` | capture, deauth, recon |
| `hashcat`, `aircrack-ng` | `offline_crack` | offline crack of captures |
| `eaphammer`, `hostapd-wpe` | `attack_wpa_enterprise_eviltwin` | 802.1X evil-twin credential capture |
| `dragondrain`, `dragontime` | `sae_dos_dragondrain`, `sae_timing_dragontime` | WPA3 / SAE checks — **functions defined but never called (dead code)**, see `docs/security.md` O6 |
| `tshark`, `airgraph-ng` | `recon_report` | uptime parse, graphs |
| `gpsd`, `termux-location`, `gpspipe` | location | GPS fix / tagging |
| `jq` | vendor KB | JSON lookup (optional) |

Module detail and gating: [`docs/modules.md`](docs/modules.md).

## Output

Everything is written under `LOG_DIR` (default `/sdcard/Download/wifi`).
The directory is made with `chmod 700`; output files use `chmod 600`.

| Path | What |
|---|---|
| `sar.jsonl` | per-AP facts, one JSON object per line |
| `cp_audit.log` | audit log: AP, method, result |
| `report_*.html` | assessment report (findings, severity, evidence) |
| `report_recon_*.html` | passive recon survey report |
| `proof/proof_*.conf` | proof of a confirmed finding: PIN, PSK, SHA-256, live-verify |
| `proof/handshake_*.cap` | captured 4-way handshake |
| `proof/pmkid_*.22000` | PMKID in hashcat 22000 form |
| `proof/eapol_*.pcap` | EAPOL capture |
| `known_bssids.txt`, `processed_bssids.txt` | state (skip already done) |
| `recon_reported.txt`, `vuln_reported.txt` | report de-dup state |

Report severities: **Critical / High / Medium / Low**.
Finding states: **confirmed / potential / beacon-based / config-based**.
Secrets are masked in the report body unless `REVEAL_CREDS=1`.
Field schema and report model: [`docs/outputs.md`](docs/outputs.md).

---

## Documentation map

| File | Content |
|---|---|
| [`docs/architecture.md`](docs/architecture.md) | components, control flow, trust boundaries, diagrams |
| [`docs/data-flow.md`](docs/data-flow.md) | scan → parse → process → report, field pipeline, scope flow |
| [`docs/modules.md`](docs/modules.md) | every assessment module, the tool it wraps, and its scope gate |
| [`docs/configuration.md`](docs/configuration.md) | all environment variables and CLI flags |
| [`docs/outputs.md`](docs/outputs.md) | `sar.jsonl` schema, proof files, report model |
| [`docs/attack-surface.md`](docs/attack-surface.md) | entry points, external dependency map, runtime diagram, what does not apply |
| [`docs/setup-and-testing.md`](docs/setup-and-testing.md) | requirements, install, run, verify commands |
| [`docs/project-structure.md`](docs/project-structure.md) | files and full function map |


---

## Author
Aliaksandr Zasinets (BITART).

## License
MIT. See [LICENSE](LICENSE).
Copyright (c) 2026 Aliaksandr Zasinets.