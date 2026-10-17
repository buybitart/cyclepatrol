# Outputs and reports

All output goes under `LOG_DIR` (default `/sdcard/Download/wifi`).
The dir is created with `chmod 700`. Data files use `chmod 600` (`run_precheck`, `save_proof`,
`sar_store_record`). Whether these modes are enforced depends on the filesystem, see
[`security.md`](security.md).

## Files

| Path | Writer | Format | Content |
|---|---|---|---|
| `sar.jsonl` | `sar_store_record` | JSON Lines | one object per AP, all parsed facts |
| `cp_audit.log` | `audit_*` / `print_log` | text | round + AP + attempt log |
| `session_<ts>_<pid>.log` | `run_cleanup` | text | copy of the session log at exit |
| `proof/proof_*.conf` | `save_proof` | `key: value` | recovered PIN / PSK / SHA-256 |
| `proof/proofs.jsonl` | `_proofs_manifest_add` | JSON Lines | chain-of-custody manifest, one line per proof file |
| `proof/handshake_*.cap` | capture modules | pcap | 4-way handshake |
| `proof/pmkid_*.22000` | PMKID modules | hashcat 22000 | PMKID hash line |
| `proof/eapol_*.pcap` | `eapol_passive_capture` | pcap | EAPOL frames |
| `report_<ts>_<pid>.html` | `make_report` | HTML | assessment report |
| `report_recon_<ts>.html` | `recon_report` | HTML | passive recon survey |
| `known_bssids.txt` | `mark_known` | text | BSSIDs with a result |
| `processed_bssids.txt` | `mark_processed` | text | BSSIDs already handled |
| `recon_reported.txt`, `vuln_reported.txt` | `report_state_add` | text | de-dup state |
| `rule_trace.log` | rule engine | text | only if `ENABLE_TRACE_RULES=1` |
| `recon_<ts>-01.csv` / `.cap` | `airodump-ng` | csv / pcap | recon capture |
| `recon_<ts>-CAPR.png` / `-CPG.png` | `airgraph-ng` | png | recon graphs (optional) |

## Data relationships (keyed by BSSID)

There is no database. State is flat files. The **BSSID** (normalized MAC) is the key that links every
record. The report joins facts, proof, and captures by BSSID.

```mermaid
erDiagram
    SAR_RECORD ||--o{ PROOF_FILE : "same bssid"
    SAR_RECORD ||--o{ CAPTURE_FILE : "same bssid"
    SAR_RECORD ||--o| STATE_LIST : "listed in"
    SAR_RECORD }o--|| REPORT : "grouped into"
    PROOF_FILE ||--|| PROOF_MANIFEST : "one line each"

    SAR_RECORD {
        string bssid PK
        string essid
        string akm_list
        string pmf_mode
        string wps_state
        string cves
    }
    PROOF_FILE {
        string bssid FK
        string pin "sensitive"
        string psk "sensitive"
        string psk_sha "integrity hash"
        string method
        string live_verified
    }
    PROOF_MANIFEST {
        string bssid FK
        string file "proof file name"
        string sha256 "hash of the proof file"
        string method
        string live_verified
    }
    CAPTURE_FILE {
        string bssid FK
        string kind "cap / 22000 / pcap"
    }
    STATE_LIST {
        string bssid "known / processed"
    }
    REPORT {
        string stamp PK
        int findings
        int criticals
    }
```

Runtime maps use the same key: `IN_SCOPE_BSSIDS`, `SESSION_PROCESSED`, `FOUND_SEEN`, `OUI_CACHE`
(see [`project-structure.md`](project-structure.md)). `pin` / `psk` are the only secret fields. They
live in the proof file only, not in `sar.jsonl` and not in the manifest.

## `sar.jsonl` schema

One JSON object per line. Keys written by `sar_store_record`, plus HE/EHT keys added by
`he_sar_enrich_json` / `eht_sar_enrich_json`, plus vendor KB keys from `vendor_kb_enrich_sar_json`.

| Group | Keys |
|---|---|
| meta | `sar_version`, `report_version`, `time` |
| identity | `bssid`, `essid`, `signal`, `channel`, `chip`, `cves` |
| WPS | `wps_ver`, `wps_state`, `device_password_id`, `config_methods`, `wps_hidden`, `sel_reg`, `pin_known`, `hidden_pin_candidate`, `cfg_methods_no_pin_display` |
| device | `maker`, `model`, `device_name`, `serial`, `firmware`, `uuid` |
| crypto | `pmf_raw`, `pmf_mode`, `pmf_grade`, `rsn_caps`, `akm_list`, `wpa3`, `wpa3_tm`, `wpa3_mode`, `owe_tm`, `rsn_gw`, `rsn_pw`, `rsn_gm`, `tkip`, `wpa1` |
| RSNXE | `rsnxe_list`, `rsnxe_cap_hex`, `transition_disable`, `td_bitmap`, `extcap_list` |
| DPP | `dpp`, `dpp_ver`, `dpp_conf`, `dpp_pkex`, `dpp_vulns` |
| Wi-Fi 6 (HE) | `he_present`, `he_bss_color`, `he_twt_supported`, `he_*_mu_mimo`, `he_*_ofdma`, `he_160mhz`, `he_spatial_streams`, `he_raw_capabilities_hex`, ... |
| Wi-Fi 7 (EHT) | `eht_present`, `eht_320mhz`, `eht_puncturing_supported`, `eht_mlo_supported`, `eht_mlo_ap`, `eht_spatial_streams`, `eht_mcs_max`, `eht_raw_capabilities_hex`, ... |

All values pass through `json_escape` before they are written.

Short example (values are examples, not from a real network):

```json
{"sar_version":"1","report_version":"5","time":"2026-10-17T12:00:00Z","bssid":"02:00:00:00:00:01","essid":"lab-ap","signal":"-58","channel":"6","chip":"realtek","cves":"CVE-2020-9395","wps_ver":"2.0","wps_state":"configured","akm_list":"PSK","pmf_mode":"opt","pmf_grade":"warn","wpa3":"NO","tkip":"NO"}
```

## Proof file (`proof_*.conf`)

Written by `save_proof` when a key is recovered. Filename:
`proof_<bssid-no-colons>_<epoch>_<pid>.conf`.

| Line | Meaning |
|---|---|
| `# time:` | ISO time |
| `# bssid:` | AP MAC |
| `# essid:` | network name |
| `# chip:` | chipset family |
| `# method:` | module that found it (`pixie`, `vendor`, `wpa_brute`, ...) |
| `# pin:` | recovered WPS PIN (or `<empty>`) |
| `# psk:` | recovered WPA key |
| `# psk_sha:` | SHA-256 of the key (integrity check for a third party) |
| `# psk_source:` | `recovered`, or `pin-as-candidate` when the PIN equals the recovered key |
| `# live_verified:` | `yes` if a live association confirmed the key |

The PSK SHA-256 lets a reviewer check the proof file was not changed, without exposing the key.
The real key is in this file. The report body can hide it (`REVEAL_CREDS`).

## Chain-of-custody manifest (`proofs.jsonl`)

`save_proof` appends one JSON line to `proof/proofs.jsonl` for every proof file it writes
(`_proofs_manifest_add`). The manifest lets a reviewer confirm the set of proofs and check that each
file is unchanged, without opening the files themselves.

| Key | Meaning |
|---|---|
| `time` | when the proof was written |
| `bssid` | AP MAC |
| `essid` | network name |
| `method` | module that recovered the key |
| `file` | proof file name |
| `sha256` | SHA-256 of the proof file (not the key) |
| `psk_sha` | SHA-256 of the key, same value as in the proof file |
| `live_verified` | `yes`, or `no` when a live check did not run |

## Report model (`make_report`)

The HTML report sections, in render order. The labels are exactly those in `make_report`. Three
sections (Channel map, Signal analysis, Diagnostic logs) are not numbered in the code.

1. Executive summary
2. Risk overview
3. Key findings in 30 seconds
4. Assets overview
- Channel map
- Signal analysis
5. Findings
6. Remediation plan
7. Positive findings
8. Evidence (Technical view)
9. Methodology and limits (Technical view)
- Diagnostic logs (Technical view)
10. Final recommendations top 5

### Simple view and Technical view

![One report, two views: a Simple view for the client and a Technical view that adds posture findings, the evidence table, methodology, and the diagnostic log](report-views.svg)

The report opens in a **Simple view** and carries a toggle (`_report_view_toggle`) that switches to a
**Technical view**. The two views share one HTML file. The toggle sets a `show-tech` class on the
page body, and the Technical-only blocks carry the `tech-only` class.

| View | Shows |
|---|---|
| Simple (default) | proof-backed Critical findings, the asset table, channel map, signal analysis |
| Technical | everything in Simple, plus posture findings, the evidence table, methodology, and the diagnostic log |

In Simple view the Findings section lists only confirmed Critical items. The posture findings (the
Medium / Low / Info beacon and config notes) move into a `tech-only` block, so the client-facing view
stays to the recovered-key facts.

### Severity (`_sev_color`)

The report palette is red / black / grey.

| Severity | Color token |
|---|---|
| Critical | `#c1121f` |
| High | `#780000` |
| Medium | `#595959` |
| Low | `#8a8a8a` |
| Info | grey |

### Finding state (`_state_badge`)

![How a finding gets its severity: Critical needs a recovered key and a non-empty proof file, otherwise the item stays a Low or Info posture note](severity-gate.svg)

| State | Meaning | Example finding |
|---|---|---|
| `confirmed` | a key was actually recovered | WPS PIN popped, WPA2 key cracked (Critical) |
| `potential` | a lead, tried but rejected | WPS known-default PIN rejected, registrar open but rejected (High) |
| `beacon` | from the beacon, not an action | PMF / 802.11w off |
| `config` | from the AKM / config | no WPA3 SAE |

Treat the finding **state** as the truth: only `confirmed` means a key was recovered. A Critical
finding is rendered only when a non-empty proof file exists for it, so Critical always has evidence
on disk.

### Proof inside a posture finding

Each posture finding (Medium / Low / Info) lists the measured beacon facts that back its rating, one
line per affected BSSID, in its Technical details. For example PMF-off shows `pmf_mode`, `pmf_raw`,
and `rsn_caps`. "no WPA3" shows `akm_list`. The legacy-cipher finding shows `tkip` and `wpa1`. These
come from the current scan, not from a template.

## Console output

During a run the phone console prints short status lines through `print_log`, in the form
`[TAG] message`. Attack and result messages are kept to a few words so a tester can read them at a
glance, for example `Got PIN.`, `key found`, `PMKID got`, `handshake got`, `proof saved`. The full
detail (BSSID, method, key hash) goes to `cp_audit.log` and the proof file, not the console.

## Return codes (`process_target`)

| Code | Name | Meaning |
|---|---|---|
| 0 | `RET_SUCCESS` | a module recovered a key |
| 1 | `RET_LOCKOUT` | AP went into WPS lockout |
| 2 | `RET_FAILED` | ran, nothing recovered |
| 3 | `RET_SKIPPED` | skipped (bad BSSID / already done) |
| 4 | `RET_ERROR` | setup error (e.g. no monitor mode) |

To get a PDF, open the HTML report in a browser and print to PDF.
