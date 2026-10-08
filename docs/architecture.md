# Architecture

CyclePatrol is a single Bash script (`cp.sh`). There is no server, no database, no network API.
It is a command-line tool. It talks to a Wi-Fi driver with `iw` / `ip`, and it calls external
wireless tools as child processes. State lives in Bash variables and in flat files under `LOG_DIR`.

## Runtime shape

- **One process, one loop.** `main` → `run_precheck` → `run_scan_loop` (endless). Child processes
  (`reaver`, `airodump-ng`, `hcxdumptool`, ...) are started and killed per task.
- **Strict mode.** `set -Eeo pipefail`, a safe `IFS` (newline + tab), `inherit_errexit`, `errtrace`.
- **Cleanup trap.** `run_cleanup` runs on `EXIT`, `INT`, `TERM`. It restores the
  interface mode, restores the original MAC, kills background PIDs, copies the session log, and
  removes the temp dir.

## Component view

```mermaid
flowchart TB
    subgraph CLI[CLI and startup]
        MAIN[main<br/>parse flags]
        VENV[validate_env<br/>input validation]
        PRE[run_precheck<br/>root, iface, tools, dirs]
        DEP[auto_install_deps<br/>apt only]
    end

    subgraph IFACE[Interface and stealth]
        MODE[ensure_mode<br/>monitor / managed]
        CHAN[iface_set_channel]
        MAC[stealth_mac_rotate<br/>macchanger / ip link]
        SCANIW[iface_do_scan<br/>iw dev scan]
    end

    subgraph DISC[Discovery]
        SB[scan_band<br/>2.4 / 5 / 6E]
        PS[parse_scan<br/>awk beacon parser]
    end

    subgraph ENRICH[Enrichment and scope]
        CHIP[get_chip / get_cves<br/>OUI -> vendor -> chipset]
        KB[vendor KB<br/>oui.json / vendors.json]
        DPP[detect_dpp_vulns]
        GATE[active_attack_permitted<br/>scope gate]
    end

    subgraph ASSESS[Assessment modules]
        WPS[WPS: pixie / vendor / null / empty / user_pin]
        CAP[Capture: pmkid / eapol / deauth handshake]
        WPA[WPA2 PSK: online brute / offline crack]
        ENT[Enterprise: eaphammer / hostapd-wpe]
    end

    subgraph OUT[Evidence and output]
        SAR[sar_store_record<br/>sar.jsonl]
        PROOF[save_proof<br/>proof_*.conf]
        REP[make_report<br/>report_*.html]
        RECON[run_recon / recon_report<br/>report_recon_*.html]
    end

    MAIN --> VENV --> PRE --> DEP
    PRE --> MODE
    MODE --> SB --> SCANIW --> PS
    PS --> CHIP --> KB
    PS --> DPP
    CHIP --> GATE
    GATE --> WPS
    GATE --> CAP
    GATE --> WPA
    GATE --> ENT
    WPS --> PROOF
    CAP --> WPA
    WPA --> PROOF
    ENT --> PROOF
    PS --> SAR
    PROOF --> REP
    SB --> RECON
```

## Per-round control flow

```mermaid
sequenceDiagram
    participant SL as run_scan_loop
    participant If as interface iw/ip
    participant Parse as parse_scan
    participant PT as process_target
    participant Gate as active_attack_permitted
    participant Tool as external tool
    participant Files as LOG_DIR

    SL->>If: stealth MAC rotate, managed
    SL->>If: scan_band 2.4 / 5 / 6E via iw scan
    If-->>SL: raw scan text
    SL->>Parse: parse_scan(raw)
    Parse-->>SL: targets, 60 fields per AP
    loop each AP up to MAX_TARGETS
        SL->>PT: process_target(fields)
        PT->>Files: sar_store_record -> sar.jsonl
        PT->>PT: build monitor_chain
        PT->>If: ensure_mode monitor
        loop each module in chain
            PT->>Gate: active_attack_permitted(bssid)?
            Note right of Gate: only active modules ask the gate
            Gate-->>PT: allow / deny
            PT->>Tool: run module: reaver / hcxdumptool / ...
            Tool-->>PT: result
            alt key recovered
                PT->>Files: save_proof -> proof_*.conf
            end
        end
        PT->>If: ensure_mode managed
    end
    SL->>Files: make_report -> report_*.html
    SL->>SL: sleep COOLDOWN
```

## Interface mode lifecycle

The adapter moves between `managed` and `monitor`. Scanning and MAC rotation happen in `managed`
(`scan_band`). Captures and active modules flip to `monitor`, then back to `managed`
(through `ensure_mode`). `run_cleanup` restores the original
type and MAC on exit.

```mermaid
stateDiagram-v2
    [*] --> Managed: run_precheck saves<br/>ORIGINAL_MAC + RESTORE_TYPE
    Managed --> Managed: stealth MAC rotate +<br/>scan_band (iw scan)
    Managed --> Monitor: ensure_mode monitor<br/>(attack / capture / recon)
    Monitor --> Managed: ensure_mode managed<br/>(after each module / AP)
    Managed --> Restore: EXIT / INT / TERM
    Monitor --> Restore: EXIT / INT / TERM
    Restore --> [*]: run_cleanup restores<br/>type + MAC, kills children
```

## Trust boundaries

```mermaid
flowchart LR
    subgraph UNTRUSTED[Untrusted - RF air]
        BEACON[Beacons / probe responses<br/>nearby APs]
    end

    subgraph OPIN[Operator input]
        ENVV[env vars: IFACE, allowlists,<br/>wordlists, external tool path]
    end

    subgraph TOOL[CyclePatrol process - root]
        VAL[validate_env / normalize_bssid<br/>/ json_escape / _html_escape]
        CORE[scan, parse, enrich, orchestrate]
    end

    subgraph DRIVER[Kernel / Wi-Fi driver]
        NL[nl80211 via iw/ip]
    end

    subgraph SUBP[External subprocesses]
        EXT[reaver, hcxdumptool, hashcat,<br/>eaphammer, airodump-ng ...]
    end

    subgraph DISK[Local files - LOG_DIR]
        O[sar.jsonl, proof_*.conf,<br/>report_*.html, logs]
    end

    BEACON -->|parsed text| VAL --> CORE
    ENVV -->|validated| VAL
    CORE --> NL
    CORE --> EXT
    CORE --> O
    EXT --> O
```

**Boundaries to note for a pentester / AppSec review:**

1. **RF air → parser.** Beacon and probe-response data is attacker-controllable input.
   It is parsed by a large `awk` program in `parse_scan`. Values that reach JSON go through
   `json_escape` and values that reach HTML go through `_html_escape`.
   BSSIDs are normalized and checked (`normalize_bssid`, `check_bssid`) before use in shell.
2. **Operator env → process.** `validate_env` checks the interface name, log dir, country code,
   numeric ranges, and the user PIN format. Allowlists and wordlists are read from env/files.
3. **Root boundary.** The tool refuses to run without root (`run_precheck`). It controls the
   Wi-Fi driver and can change MAC, mode, and regulatory domain.
4. **Subprocess boundary.** External tools do the real RF work. CyclePatrol builds their arguments
   from parsed/validated values.
5. **Disk boundary.** Recovered keys are stored on disk (see [`outputs.md`](outputs.md) and
   [`security.md`](security.md)).

## Why these choices (as seen in code)

- **Fail-safe on missing tools.** Each module checks `command -v <tool>` and returns cleanly if the
  tool is absent. Optional tools only print a warning in `run_precheck`.
- **Interface restore.** `ORIGINAL_MAC` and `RESTORE_TYPE` are saved in `run_precheck` and restored
  in `run_cleanup`, so the adapter is left in a clean state.
- **De-dup state.** `processed_bssids.txt` / `known_bssids.txt` stop repeated work on the same AP.
