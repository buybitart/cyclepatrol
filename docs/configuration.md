# Configuration

CyclePatrol has **no config file**. All config is through **environment variables** and **CLI flags**.
Defaults are set at the top of `cp.sh` with `: "${VAR:=default}"`.
Numeric variables are range-checked in `validate_env`. A bad value either falls back to
the default or stops the tool (`FATAL`).

## CLI flags (`main`)

| Flag | Effect |
|---|---|
| (none) | one-click: auto-install deps, one recon survey, then scan loop |
| `--recon` | passive recon survey only (airodump-ng), write `report_recon_*.html`, exit |
| `--recon-secs <n>` | recon capture seconds (5..3600, default 60) |
| `--recon-gps` | enable GPS tagging in recon (needs gpsd) |
| `--no-recon` | skip the startup recon survey |
| `--no-install` | do not auto-install tools (`AUTO_INSTALL=0`) |
| `--baseline-save [path]` | save a baseline from SAR / rule results |
| `--baseline-compare <a.bl> <b.bl>` | compare two baselines |
| `--from-sar <sar.jsonl>` | SAR input for `--baseline-save` |
| `--gps-test` | test the Android→Kali GPS bridge, print one fix, exit |
| `--trace-rules` | diagnostic rule-engine trace |
| `-h`, `--help` | print usage and exit |

## Core

| Variable | Default | Meaning |
|---|---|---|
| `IFACE` | `wlan2` | Wi-Fi interface (regex-checked) |
| `LOG_DIR` | `/sdcard/Download/wifi` | output dir (no `..`, no leading `-`) |
| `HASH_DIR` | `<LOG_DIR>/proof` | proof files dir |
| `REG_DOMAIN` | `PL` | regulatory domain (2 letters) |
| `SCAN_TIMEOUT` | `25` | per-scan timeout s (5..3600) |
| `COOLDOWN` | `5` | sleep between rounds s |
| `MAX_TARGETS` | `20` | max APs per round (1..1000) |
| `MAX_CORES` | `4` | worker cap (1..64) |
| `SCAN_24` / `SCAN_5` / `SCAN_6E` | `1` / `1` / `1` | scan 2.4 / 5 / 6 GHz |
| `RSSI_ATTACK_MIN` | `-75` | skip APs weaker than this dBm |
| `AUTO_INSTALL` | `1` | auto-install missing tools via apt |
| `ENABLE_DRY_RUN` | `0` | skip real actions where supported |

## Stealth / interface

| Variable | Default | Meaning |
|---|---|---|
| `ENABLE_STEALTH_MAC` | `1` | random MAC each round |
| `MAC_ROTATE_DELAY` | `3` | seconds after MAC change |
| `ENABLE_STEALTH_WPS_ID` | `1` | random WPS UUID / identity |
| `IFACE_RETRIES` / `IFACE_RETRY_DELAY` | `3` / `2` | mode-switch retries |

## WPS

| Variable | Default | Meaning |
|---|---|---|
| `USER_PROVIDED_PIN` | (empty) | one 8-digit PIN for `attack_user_pin` |
| `PIXIE_TIMEOUT` | `20` | pixie step timeout s |
| `BASE_LOCK_DELAY` / `MAX_LOCK_DELAY` | `5` / `7` | WPS lockout backoff |
| `PIXIEWPS_FALLBACK_MODE` | (empty) | force a pixiewps mode |

## Capture / crack

| Variable | Default (top of file) | `--help` says | Meaning |
|---|---|---|---|
| `ENABLE_PMKID` | `1` | 1 | passive PMKID |
| `PMKID_TIMEOUT` | `10` | 10 | PMKID capture s |
| `ENABLE_EAPOL_PASSIVE` | `1` | 1 | passive EAPOL |
| `EAPOL_PASSIVE_TIMEOUT` | `30` | 30 | EAPOL window s |
| `ENABLE_PMKID_ACTIVE` | `1` | 0 | active PMKID (gated) |
| `ENABLE_DEAUTH` | `1` | 0 | deauth handshake (gated) |
| `ENABLE_DEAUTH_BROADCAST` | `1` | 0 | broadcast deauth (DoS-class) |
| `DEAUTH_BURST` / `DEAUTH_MAX_CLIENTS` | `5` / `3` | same | deauth tuning |
| `DEAUTH_CAPTURE_SECS` / `DEAUTH_CLIENT_SCAN_SECS` | `20` / `6` | same | windows |
| `ENABLE_WPA_BRUTE` | `1` | 1 | online WPA2-PSK dict (gated) |
| `WPA_BRUTE_MAX` / `WPA_BRUTE_TRY_SECS` | `30` / `8` | same | brute bounds |
| `WPA_BRUTE_WORDLIST` | (empty) | — | passphrase file |
| `ENABLE_OFFLINE_CRACK` | `1` | — | offline crack of captures |
| `OFFLINE_CRACK_WORDLIST` / `OFFLINE_CRACK_MAX_SECS` | (empty) / `30` | — | crack input / budget |
| `HASHCAT_TIMEOUT` / `HASHCAT_OPTS` | `0` / (empty) | — | hashcat tuning |
| `ENABLE_DEFAULTKEY_GEN` | `1` | — | vendor-default key candidates |
| `ENABLE_LIVE_VERIFY` / `LIVE_VERIFY_SECS` | `1` / `15` | — | live association check |

> Note the **default mismatch**: the top-of-file `:=` defaults set `ENABLE_PMKID_ACTIVE`,
> `ENABLE_DEAUTH`, `ENABLE_DEAUTH_BROADCAST` to `1`, while `--help` and the `validate_env`
> fallbacks describe them as `0` / fail-closed. The top-of-file value takes effect. See
> [`security.md`](security.md).

## WPA3 / SAE / DPP

| Variable | Default | Meaning |
|---|---|---|
| `ENABLE_DRAGONBLOOD_POSTURE` | `1` | WPA3 posture notes in recon report |
| `ENABLE_DPP_SNIFF` / `DPP_SNIFF_SECS` | `1` / `8` | DPP sniff (used in `process_target`) |

**No effect in this build (dead code):** `ENABLE_SAE_PROBE`, `ENABLE_SAE_DOS`, `ENABLE_SAE_TIMING`,
`SAE_DOS_SECS`, `SAE_TIMING_SECS`, `DRAGON_DOS_TOOL`, `DRAGON_TIME_TOOL`. The SAE DoS/timing
functions that read them (`sae_dos_dragondrain`, `sae_timing_dragontime`) are never called, and no
SAE-probe function exists. See [`modules.md`](modules.md#sae-dos--timing--present-but-not-called-dead-code).

## Enterprise (802.1X)

| Variable | Default | Meaning |
|---|---|---|
| `ENABLE_EVILTWIN` | `1` | evil-twin EAP capture (gated) |
| `EVILTWIN_TOOL` | (auto) | `eaphammer` or `hostapd-wpe` |
| `EVILTWIN_TIMEOUT` | `120` | rogue-AP runtime s (20..1800) |
| `EVILTWIN_ESSID_ALLOWLIST` / `_FILE` | (unset) | ESSIDs allowed for evil-twin (required) |
| `EVILTWIN_DEAUTH` | `0` | deauth to push clients |
| `ENABLE_WPA_AUDIT` | `0` | external/passive WPA audit |
| `WPA_AUDIT_MODE` | `passive` | `passive` or `external` |
| `WPA_AUDIT_EXTERNAL_TOOL` | `./wpa_audit_target.sh` | external script (not in repo) |
| `WPA_AUDIT_WORDLIST` | (empty) | wordlist for external audit |

## Scope / disclosure

| Variable | Default | Meaning |
|---|---|---|
| `ALLOW_ALL_TARGETS` | `1` | `1` → allowlist state = `allow_all` |
| `SCOPE_ALL_AUTHORIZED` | `0` (reset later in the script) | `1` → gate allows all |
| `ACTIVE_BSSID_ALLOWLIST` | (unset) | CSV of allowed BSSIDs |
| `ACTIVE_ALLOWLIST_FILE` | (unset) | file of allowed BSSIDs |
| `SCOPE_ESSID_PREFIX` | (empty) | limit evil-twin by ESSID prefix |
| `REVEAL_CREDS` | `1` | `1` → show PIN/PSK in report + logs |

## Location / war-driving

| Variable | Default | Meaning |
|---|---|---|
| `ENABLE_LOCATION` | `no` | enable location sampling |
| `LOCATION_PROVIDER` | (auto) | `gps` / `network` |
| `LOCATION_TIMEOUT` | `20` | fix timeout s |
| `LOCATION_NMEA_PORT` | `2947` | gpsd/NMEA port |
| `RECON_GPS` / `RECON_GPSD_SRC` | `0` / `udp://*:2947` | recon GPS tagging |

## Reporting / diagnostics

| Variable | Default | Meaning |
|---|---|---|
| `ENABLE_REPORT_DEDUP` | `1` | report each BSSID once |
| `ENABLE_VENDOR_KB` / `VENDOR_DIR` | `1` / `./vendor` | vendor KB lookups |
| `ENABLE_TRACE_RULES` | `0` | write `rule_trace.log` |
| `ENABLE_SURVEY` / `ENABLE_ASSESSMENT_RESULT` | `yes` / `yes` | survey + result records |
| `ENABLE_VALIDATION` | `no` | self-test with mock inputs |
| `ENABLE_MOBILE_REEVAL` | `1` | re-check a BSSID if RSSI improves |
| `REEVAL_RSSI_IMPROVE_DB` / `REEVAL_TTL_SEC` | `6` / `300` | re-eval thresholds |
| `SKIP_PROCESSED` | `1` | skip already-done BSSIDs |

