# Project structure

## Repository tree

```text
wps_scan/
├── cp.sh              # the whole tool (bash, 7130 lines, 159 functions)
├── vendor/
│   ├── oui.json       # OUI prefix -> vendor id map (large)
│   └── vendors.json   # vendor knowledge base: name, behaviours, recommendations
├── README.md
└── LICENSE            # MIT, (c) 2026 Aliaksandr Zasinets
```

There is no `docs/` in the original repo (this documentation adds it), no test suite, no CI config,
and no separate config file. Config is env vars + CLI flags only.

## Vendor data files

| File | Format | Used by | Content |
|---|---|---|---|
| `vendor/oui.json` | JSON object `{ "oo:uu:ii": "vID" }` | `vendor_kb_lookup_oui` | map a MAC OUI to a vendor id |
| `vendor/vendors.json` | JSON object keyed by vendor id | `vendor_kb_vendor_name` / `_array` | `name`, `behaviours`, `recommendations` per vendor |

Lookups prefer `jq` if present, else fall back to `grep`/`awk`/`sed`.
Controlled by `ENABLE_VENDOR_KB` (default 1) and `VENDOR_DIR` (default `./vendor`).

## Function map (by area)

159 functions. Grouped by job, with a few line anchors.

### Startup / config / validation
`main`, `validate_env`, `validate_positive_int`, `run_precheck`,
`auto_install_deps`, `_pkg_for`, `check_bssid`.

### Logging / audit / cleanup
`ts_now`, `audit_write` / `audit_blank` / `audit_header`, `print_log`,
`log_target_detail`, `log_attempt`, `run_cleanup`.

### Interface / mode / stealth
`iface_get_type`, `iface_set_type`, `ensure_mode`, `iface_set_channel`,
`iface_get_mac`, `iface_do_scan`, `iface_do_scan_freq`,
`chan2freq`, `freq2chan`,
`stealth_gen_mac`, `stealth_gen_uuid`, `stealth_mac_apply`,
`stealth_mac_rotate`, `stealth_wps_id_block`.

### Normalize / PIN math
`normalize_bssid`, `normalize_oui`, `pin_check`, `wps_checksum`,
`pin_from_mac`, `easybox_pin`, `pin_entropy_class`.

### Scan / parse
`scan_band`, `parse_scan` with inner awk `reset` / `row` / `emit`,
`run_scan_loop`.

### Enrichment / vendor KB
`get_vendor`, `get_chip`, `get_cves`,
`vendor_kb_available`, `vendor_kb_lookup_oui`, `vendor_kb_vendor_name`,
`vendor_kb_vendor_array`, `vendor_kb_enrich_sar_json`,
`he_sar_enrich_json`, `eht_sar_enrich_json`.

### Scope / allowlist / authorization
`allowlist_add_token`, `allowlist_load_file`, `allowlist_load_csv`,
`allowlist_init_active`, `active_attack_permitted`,
`eviltwin_essid_add_token`, `eviltwin_allowlist_init`,
`eviltwin_essid_allowlisted`, `is_wpa_enterprise`.

### Lockout / state / backoff
`lock_state`, `in_lockout`, `already_known`, `is_processed`,
`mark_processed`, `mark_known`, `do_backoff`,
`measure_lockout_strength`, `wait_socket`.

### Assessment modules
`attack_user_pin`, `attack_pixie`, `attack_vendor`, `attack_null`,
`attack_empty`, `attack_pmkid`, `eapol_passive_capture`,
`attack_pmkid_active`, `attack_deauth_handshake`,
`attack_wpa_enterprise_eviltwin`, `attack_wpa_psk_brute`,
`detect_dpp_vulns`, `dpp_sniff_monitor`, `_dragonblood_from_auth`.
Support: `_reaver_init_help_cache`, `_reaver_has_opt`,
`extract_wps_hex`, `validate_wps_params`.

### Offline crack
`_resolve_wordlist`, `_hashcat_22000`, `_aircrack_cap`, `_run_cracker`,
`_defaultkey_list`, `offline_crack`.

### WPA audit
`wpa_audit_allowed`, `wpa_audit_posture`, `wpa_audit_external_dispatch`.

### Evidence / records
`report_state_*`, `sar_json_get`, `sar_store_record`,
`sar_store_locked_stub`, `save_registrar`, `verify_psk_live`,
`save_proof`, `_write_finding_proof`, `_iso_write_proof`.

### Location / war-driving
`location_is_enabled`, `location_ingest_scan_targets`, `location_env`,
`location_valid`, `_dd_to_nmea`, `_nmea_checksum`, `_nmea_send`,
`_loc_parse_termux`, `location_fix`, `location_nmea_feeder`.

### Reporting
`make_report` with helpers `_sar_field`, `_band_from_ch`, `_html_escape`,
`_attempts_for`, `_evtype_for`, `_finding_meta`, `_proof_for`,
`_show_pin` / `_show_psk` / `_mask_psk`, `_sev_color` / `_sev_bg` / `_sev_label`,
`_state_badge`, `_prio_badge`, `_fcard`.
Recon: `run_recon`, `recon_report`, `_recon_uptime_fmt`, `_recon_embed_png`.

### Baseline
`baseline_cli_save`, `baseline_cli_compare`.

### Defined but not called (dead code)

Verified by usage audit — these functions have no caller and are not dispatched dynamically:
`sae_dos_dragondrain`, `sae_timing_dragontime`, `sar_store_locked_stub`,
`sar_json_get`, `rule_result_pack`, `rule_result_emit_wire`,
`_write_finding_proof`, `_iso_write_proof`, `_attempts_for`,
`_wps_tried_rejected`, `get_vendor_lock_type`.
They are listed in the groups above by purpose, but do not run in this build. See
[`security.md`](security.md) O6. Note: `_ar_engine_dw` is a dynamic hook that only ever
calls `ar_dual_write_after_scanner`, which is not defined — so it is a no-op.

## Globals and state

- Band channel lists: `BAND_24`, `BAND_5`, `BAND_6E_PSC`.
- Arrays: `FOUND_TARGETS`, `SAR_RECORDS`, `RULE_RESULTS`, `LOCATION_OBSERVATIONS`, `CREDS_FOUND`,
  `ATTEMPT_LOG`, `BG_PIDS`.
- Maps: `SESSION_PROCESSED`, `FOUND_SEEN`, `ACTIVE_ALLOWLIST`, `EVILTWIN_ALLOWLIST`,
  `IN_SCOPE_BSSIDS`, `OUI_CACHE`.
- File path vars set in `run_precheck`: `TMP_DIR`, `LOG_SESSION`, `LOG_AUDIT`, `SCAN_RAW`,
  `SCAN_TARGETS`, `LOCK_FILE`, `KNOWN_FILE`, `PROCESSED_FILE`, `REPORT_STATE_*`.
- Static data maps: `OUI_TO_VENDOR`, `VENDOR_TO_LIKELY_CHIPSET`, `CHIPSET_CVES`,
  `CHIPSET_PIXIE_MODES`, `DPP_CVES`, `KNOWN_PINS_DB`.
