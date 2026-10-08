# Data flow

This page follows the data from the air to the report.

## End-to-end

```mermaid
flowchart LR
    A[RF air<br/>beacons] --> B[iw dev scan<br/>raw text]
    B --> C[parse_scan awk]
    C --> D["targets file<br/>60 fields per AP<br/>sep = 0x1E"]
    D --> E[process_target]
    E --> F[enrich:<br/>chipset, CVE, PMF,<br/>WPA3 mode, DPP, HE/EHT]
    F --> G[sar_store_record]
    G --> H[(sar.jsonl)]
    E --> I[monitor_chain<br/>assessment modules]
    I --> J{key recovered?}
    J -- yes --> K[save_proof]
    K --> L[(proof_*.conf)]
    J -- no --> M[attempt log only]
    H --> N[make_report]
    L --> N
    M --> N
    N --> O[(report_*.html)]
```

## 1. Scan

- `run_scan_loop` calls `scan_band` for each enabled band: `2.4` (default on), `5` (on), `6E` (on).
  Channel lists: `BAND_24`, `BAND_5`, `BAND_6E_PSC`.
- `scan_band` turns channels into frequencies (`chan2freq`) and runs
  `iw dev <iface> scan freq ...` with a `SCAN_TIMEOUT` (default 25s). Output is appended to a raw file.
- Before each round, if `ENABLE_STEALTH_MAC=1`, the MAC is rotated in managed mode.

## 2. Parse

- `parse_scan` is a large `awk` program. It reads the raw `iw scan` text and builds **one line per
  AP** with about **60 fields**, joined by the ASCII record separator `0x1E`.
- Fields parsed from the beacon / RSN / RSNXE / WPS / vendor elements, for example:
  - identity: `bssid`, `essid`, `signal`, `channel`, `freq`
  - WPS: `wps_state`, `wps_ver`, `device_password_id` (`dpid`), `config_methods`, `wps_hidden`, `sel_reg`,
    device info (`maker`, `model`, `fw`, `devname`, `serial`, `uuid`)
  - crypto: `akm_list`, `pmf`, `rsn_caps`, `rsn_gw/pw/gm` ciphers, `tkip`, `wpa1`, `wpa3`, `owe`
  - RSNXE: `rsnxe_list`, `rsnxe_cap_hex` (SAE-H2E, SAE-PK, Trans-Disable, PMF bits, ...)
  - DPP: `dpp`, `dpp_ver`, `dpp_conf`, `dpp_pkex`
  - Wi-Fi 6 (HE) and Wi-Fi 7 (EHT): capability flags and raw hex

## 3. Process per AP

`process_target` runs for each parsed AP (up to `MAX_TARGETS`, default 20):

```mermaid
flowchart TD
    S[fields in] --> NB[normalize_bssid + check_bssid]
    NB --> SK{already processed<br/>or known?}
    SK -- yes --> SKIP[skip]
    SK -- no --> MARK[mark processed]
    MARK --> ENR[enrich: get_chip, get_cves,<br/>PMF grade, wpa3_mode,<br/>detect_dpp_vulns, transition-disable]
    ENR --> SAR[sar_store_record -> sar.jsonl]
    SAR --> SCOPE[add BSSID to IN_SCOPE_BSSIDS]
    SCOPE --> CHAIN[build monitor_chain]
    CHAIN --> MON[ensure_mode monitor]
    MON --> RUN[run each module in order]
    RUN --> RES{success?}
    RES -- yes --> OK[proof + return RET_SUCCESS]
    RES -- lockout --> LK[measure_lockout_strength + stop]
    RES -- no --> NXT[next module]
    NXT --> POST[WPA audit posture / eviltwin / wpa psk brute]
```

The module order (`monitor_chain`) is decided from the AP facts:

- If the AP needs a user PIN (`dpid=user`): `attack_user_pin` then `attack_pixie`.
- Else: `attack_pixie`, and for WPS 1.0 / 2.x or a known chip family also `attack_null`, `attack_empty`.
- Then, if enabled: `attack_pmkid`, `attack_pmkid_active`, `eapol_passive_capture`,
  `attack_deauth_handshake`, and `attack_vendor` (when not user-PIN).
- After the monitor chain: `wpa_audit_posture`, optional external WPA audit, enterprise evil-twin
  (if AKM is 802.1X), and WPA2-PSK brute (if AKM is PSK).

### How the chain is built (`process_target`)

```mermaid
flowchart TD
    A[AP facts] --> B{dpid = user?}
    B -- yes --> C[attack_user_pin, attack_pixie]
    B -- no --> D[attack_pixie]
    D --> E{WPS 1.0/2.x<br/>or known chip?}
    E -- yes --> F[+ attack_null, attack_empty]
    E -- no --> G[skip null/empty]
    C --> H{enabled flags}
    F --> H
    G --> H
    H -- ENABLE_PMKID --> I[+ attack_pmkid]
    H -- ENABLE_PMKID_ACTIVE --> J[+ attack_pmkid_active]
    H -- ENABLE_EAPOL_PASSIVE --> K[+ eapol_passive_capture]
    H -- ENABLE_DEAUTH --> L[+ attack_deauth_handshake]
    H -- not user-PIN --> M[+ attack_vendor]
    I --> RUN[run chain in monitor mode<br/>stop on success or lockout]
    J --> RUN
    K --> RUN
    L --> RUN
    M --> RUN
    RUN --> POST[then: wpa_audit_posture,<br/>eviltwin if 802.1X,<br/>wpa psk brute if PSK]
```

## 4. Scope decision

```mermaid
flowchart TD
    Q[active module wants BSSID b] --> G{active_attack_permitted b}
    G --> C1{b in IN_SCOPE_BSSIDS?}
    C1 -- yes --> ALLOW[allow]
    C1 -- no --> C2{SCOPE_ALL_AUTHORIZED = 1?}
    C2 -- yes --> ALLOW
    C2 -- no --> C3{allowlist state = allow_all?<br/>ALLOW_ALL_TARGETS=1}
    C3 -- yes --> ALLOW
    C3 -- no --> C4{b in ACTIVE_ALLOWLIST?}
    C4 -- yes --> ALLOW
    C4 -- no --> DENY[deny]
```

Note: `process_target` adds every scanned BSSID to `IN_SCOPE_BSSIDS` **before** the modules run.
So the first check is already true for any AP the scanner saw. See [`security.md`](security.md)
for the default-posture impact. Only active modules call the gate; the WPS PIN chain and the
passive PMKID/EAPOL capture do not (see [`modules.md`](modules.md)).

## 5. Evidence and report

- `sar_store_record` writes one JSON line per AP to `sar.jsonl` (chmod 600).
- `save_proof` writes a proof file when a key is recovered: PIN, PSK, PSK SHA-256, method, and a
  `live_verified` marker (from `verify_psk_live`, a one-shot `wpa_supplicant` association).
- `make_report` builds the HTML report from the SAR records, proof files, and attempt log.

## Field transport detail

| Stage | Form | Separator / format |
|---|---|---|
| `iw scan` → parser | text | native `iw` output |
| parser → loop | targets file | field sep `0x1E`, one AP per line |
| loop → `sar.jsonl` | JSON | one object per line (JSONL) |
| finding → proof | `key: value` text | `proof_*.conf` |
| report | HTML | `report_*.html` |

## Example runtime: deauth-assisted handshake → crack

One active module, end to end (`attack_deauth_handshake` → `offline_crack`).
It shows the scope gate, the monitor-mode flip, and proof writing.

```mermaid
sequenceDiagram
    participant PT as process_target
    participant DH as attack_deauth_handshake
    participant Gate as active_attack_permitted
    participant If as interface (iw/ip)
    participant AD as airodump-ng
    participant AR as aireplay-ng / mdk4
    participant OC as offline_crack
    participant HC as hashcat / aircrack-ng
    participant F as LOG_DIR

    PT->>DH: run (bssid, channel)
    DH->>Gate: active_attack_permitted(bssid)?
    alt denied
        Gate-->>DH: deny
        DH-->>PT: skip (log_attempt FAIL)
    else allowed
        Gate-->>DH: allow
        DH->>If: ensure_mode monitor
        DH->>AD: capture on channel (pcap + csv)
        DH->>AR: deauth burst to client(s)
        AR-->>AD: client reconnects -> 4-way handshake
        AD-->>DH: capture file hs_*.cap
        DH->>F: save_proof handshake_*.cap
        DH->>OC: offline_crack(cap)
        OC->>HC: hashcat -m 22000 / aircrack (if wordlist hits)
        HC-->>OC: key or no key
        alt key found
            OC->>F: proof_*.conf (psk + sha256)
        end
        DH->>If: ensure_mode managed
    end
```

`ENABLE_DEAUTH_BROADCAST=1` lets the deauth go broadcast when no client is found this is
DoS-class. Offline crack only recovers a key if the passphrase is in the wordlist.
