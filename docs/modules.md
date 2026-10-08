# Assessment modules

This page lists the assessment modules in `cp.sh`. Each module wraps a standard wireless tool.
This is a **map of the code**, not a how-to. No attack steps or payloads are given here.

Columns:
- **Gate** = does the module call `active_attack_permitted()` before it runs?
- **Enable var** = environment variable that turns it on/off.
- **On success** = the finding type written if a key is recovered.

## WPS modules

| Module | Wraps | Gate | Enable var | Note |
|---|---|---|---|---|
| `attack_pixie` | `reaver` `-K`, or `wpa_supplicant`+`wpa_cli`, `pixiewps` | no | always in chain | WPS offline PIN (Pixie Dust). Uses `CHIPSET_PIXIE_MODES` per chip. On success → `pixie` (Critical). |
| `attack_vendor` | `reaver` `-p` | no | in chain when not user-PIN | Tries PINs from `KNOWN_PINS_DB` (OUI → default PIN list) and derived PINs (`pin_from_mac`, `easybox_pin`). On success → `vendor` (Critical). |
| `attack_null` | `reaver` `-p 00000000` | no | WPS 1.0/2.x or known chip | All-zero PIN check. On success → `null` (Critical). |
| `attack_empty` | `reaver` `-p ""` | no | WPS 1.0/2.x or known chip | Empty PIN check. On success → `empty` (Critical). |
| `attack_user_pin` | `reaver` `-p <PIN>` | no | `USER_PROVIDED_PIN` set | Tries one operator-supplied 8-digit PIN. On success → `user_pin` (High). |

Support: `pin_check` / `wps_checksum` (WPS PIN checksum), `extract_wps_hex` / `validate_wps_params`
(read M1..M3 hex from the `wpa_supplicant` log for `pixiewps`), `measure_lockout_strength` and
`in_lockout` (WPS lockout handling with backoff).

## Capture modules

| Module | Wraps | Gate | Enable var | Note |
|---|---|---|---|---|
| `attack_pmkid` | `hcxdumptool`, `hcxpcapngtool` | no | `ENABLE_PMKID` (1) | Passive PMKID capture → hashcat `22000` file. |
| `eapol_passive_capture` | `tcpdump` (`ether proto 0x888e`) | no | `ENABLE_EAPOL_PASSIVE` (1) | Passive EAPOL collection. |
| `attack_pmkid_active` | `hcxdumptool` | **yes** | `ENABLE_PMKID_ACTIVE` | Active PMKID solicitation (PMF-resilient). |
| `attack_deauth_handshake` | `airodump-ng` + `aireplay-ng`/`mdk4` | **yes** | `ENABLE_DEAUTH`, `ENABLE_DEAUTH_BROADCAST` | Deauth-assisted 4-way handshake capture. Broadcast deauth is DoS-class. |

## WPA2-PSK key recovery

| Module | Wraps | Gate | Enable var | Note |
|---|---|---|---|---|
| `offline_crack` | `hashcat -m 22000` / `aircrack-ng` (+ `hcxpcapngtool`) | no (runs on own captures) | `ENABLE_OFFLINE_CRACK` (1) | Crack captured PMKID / handshake. `_defaultkey_list` can build a vendor-default candidate list. On success → `pmkid` / `handshake` / `defaultkey` (Critical). |
| `attack_wpa_psk_brute` | `wpa_supplicant` (online association) | **yes** | `ENABLE_WPA_BRUTE` (1) | Online dictionary, bounded by `WPA_BRUTE_MAX` / `WPA_BRUTE_TRY_SECS`. On success → `wpa_brute` (Critical). |
| `verify_psk_live` | `wpa_supplicant`, `wpa_passphrase` | **yes** | `ENABLE_LIVE_VERIFY` (1) | One-shot association to confirm a recovered key is real. |

## Enterprise (802.1X)

| Module | Wraps | Gate | Enable var | Note |
|---|---|---|---|---|
| `attack_wpa_enterprise_eviltwin` | `eaphammer` or `hostapd-wpe` | **yes** | `ENABLE_EVILTWIN` | Runs only when AKM is enterprise (`is_wpa_enterprise`). Also needs an **ESSID allowlist** (`EVILTWIN_ESSID_ALLOWLIST` / `_FILE`). `eaphammer` needs a one-time cert (`--cert-wizard`). On capture → `eviltwin_eap`. |
| `wpa_audit_posture` | internal | **yes** | `ENABLE_WPA_AUDIT` | Posture note only (passive mode). |
| `wpa_audit_external_dispatch` | external script `WPA_AUDIT_EXTERNAL_TOOL` | via posture | `WPA_AUDIT_MODE=external` | Default path is `./wpa_audit_target.sh`, which is **not in the repo** — operator must supply it. |

## WPA3 / SAE and DPP

| Module | Wraps | Gate | Enable var | Note |
|---|---|---|---|---|
| `detect_dpp_vulns` | internal (reads scan) | no | `dpp=YES` AP | Called in `process_target`. Maps DPP state to `DPP_CVES` references. |
| `dpp_sniff_monitor` | `iw` monitor sniff | no | `ENABLE_DPP_SNIFF` (1) | Called in `process_target` when DPP risk is seen. |
| `_dragonblood_from_auth` | internal | no | `ENABLE_DRAGONBLOOD_POSTURE` (1) | WPA3 posture note used by `recon_report`. |

### SAE DoS / timing — present but NOT called (dead code)

`sae_dos_dragondrain` (wraps `dragondrain`) and `sae_timing_dragontime` (wraps
`dragontime`) are **defined but never called** anywhere in `cp.sh` (verified: the only lines that
name them are their own definitions). So in this build:

- these two WPA3/SAE checks never run;
- `ENABLE_SAE_DOS`, `ENABLE_SAE_TIMING`, `SAE_DOS_SECS`, `SAE_TIMING_SECS`, `DRAGON_DOS_TOOL`,
  `DRAGON_TIME_TOOL` have **no runtime effect** (they are only read inside those uncalled functions);
- `ENABLE_SAE_PROBE` is read only by `validate_env` and `--help`; there is **no SAE-probe function**.

The `active_attack_permitted` calls inside these functions therefore never execute.

## Gate summary

- **Gated and actually reachable (ask `active_attack_permitted`):** active PMKID, deauth handshake,
  WPA2-PSK online brute, live PSK verify, enterprise evil-twin, WPA audit.
- **Not gated (run on any scanned AP):** WPS chain (`pixie`, `vendor`, `null`, `empty`, `user_pin`),
  passive PMKID, passive EAPOL, offline crack (works on captures already taken).
- **Gated but dead code (never called):** SAE DoS, SAE timing (see note above).

`active_attack_permitted` is called by the six reachable active modules listed above. Two further
calls sit inside the uncalled SAE functions, so they never run.
See [`security.md`](security.md) for why the gate split matters.

## Chipset / vendor data used by modules

| Map | Use |
|---|---|
| `OUI_TO_VENDOR` | OUI prefix → vendor |
| `VENDOR_TO_LIKELY_CHIPSET` | vendor → chipset family |
| `CHIPSET_CVES` | chipset → CVE hints shown in output |
| `CHIPSET_PIXIE_MODES` | chipset → `pixiewps --mode` |
| `DPP_CVES` | DPP issue → reference string |
| `KNOWN_PINS_DB` | OUI → list of known default WPS PINs |

`vendor/oui.json` and `vendor/vendors.json` extend this at runtime (`ENABLE_VENDOR_KB=1`).
See [`outputs.md`](outputs.md) and [`project-structure.md`](project-structure.md).
