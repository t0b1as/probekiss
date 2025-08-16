Kismet Probe Monitor

This repo contains two Python scripts that work together to discover Wi-Fi probe requests in Kismet logs while ignoring “known” networks and clients:

build_known_ssids.py — builds a list of known SSIDs and known clients (MACs) from your Kismet logs and writes a single reference file.

monitor_probes.py — continuously scans Kismet logs for probe requests and writes an aggregate JSON log for unknown SSIDs/clients, organized by source file → SSID → clients.

Both scripts use only Python’s standard library and support modern Kismet SQLite exports (.kismetdb, .kismet) and the legacy XML export (.netxml).

Table of Contents

Requirements

Input Files

Output Files

Produced by build_known_ssids.py

Produced by monitor_probes.py

Script 1: build_known_ssids.py

What it does

CLI

Usage

How “known” is detected

Script 2: monitor_probes.py

What it does

Aggregate JSON structure

CLI

Usage

Filtering logic

Notes & Behaviors

Troubleshooting

Requirements

Python 3.7+

No external packages required.

Run the scripts from the folder that contains your Kismet logs (or point your shell there).

Input Files

Place one or more of the following Kismet exports in the working directory:

*.kismetdb or *.kismet — Kismet SQLite databases.

*.netxml — legacy Kismet XML export (optional; used as a fallback).

The scripts dynamically inspect the database schema and JSON blobs inside Kismet’s devices table (and, if needed, other tables) to extract SSIDs/clients.

Output Files
Produced by build_known_ssids.py

known_entities.json
A single reference file with known SSIDs and known clients (MAC addresses):

{
  "ssids": ["HomeWiFi", "Cafe_WLAN", "..."],
  "clients": ["A0:B1:C2:D3:E4:F5", "00:11:22:33:44:55", "..."]
}


Backward-compatibility: If you already have a legacy known_ssids.json (a JSON list or a dict with ssids/clients), monitor_probes.py can still use it as a fallback.

Produced by monitor_probes.py

unknown_probes.json (aggregate log; continuously updated)
Hierarchical structure: source file → SSID → clients.
Example:

{
  "capture1.kismetdb": {
    "source_mtime": 1723802330,
    "source_size": 12345678,
    "ssids": {
      "Cafe_WLAN": {
        "total_count": 3,
        "first_seen": "2025-08-16T11:00:03+0200",
        "last_seen":  "2025-08-16T11:02:41+0200",
        "clients": {
          "A0:B1:C2:D3:E4:F5": {
            "count": 2,
            "first_seen": "2025-08-16T11:00:03+0200",
            "last_seen":  "2025-08-16T11:02:41+0200",
            "timestamps": "2025-08-16T11:00:03+0200,2025-08-16T11:02:41+0200"
          }
        }
      }
    }
  }
}


Field meanings

source_mtime, source_size: file metadata captured when the source was scanned.

total_count: total number of probe events for that SSID (from this source).

first_seen, last_seen: ISO-8601 timestamps with local offset.

clients[MAC].timestamps: single line, comma-separated list of timestamps for that client/SSID pair.

Script 1: build_known_ssids.py
What it does

Scans all .kismetdb/.kismet/.netxml files in the current directory to collect:

Known SSIDs — beaconed/advertised SSIDs (and OWE SSIDs where present).

Known clients — device MACs that clearly behave like 802.11 clients (see heuristics below).

Writes both sets to known_entities.json.

CLI
python3 build_known_ssids.py [--output known_entities.json] [--verbose]


--output (default: known_entities.json): output JSON file.

--verbose: print diagnostic messages (counts per file, fallbacks used, etc.).

Usage
# from the directory containing your Kismet logs
python3 build_known_ssids.py --verbose
# -> writes known_entities.json (includes ssids + clients)

How “known” is detected

SSIDs: Prefer dot11.device/dot11.advertised_ssid_map (Kismet JSON), fall back to common SSID fields, and to <essid> in .netxml (non-cloaked).

Clients: A device is considered a client (and its devmac recorded) if its JSON indicates probe behavior (e.g., dot11.probed_ssid_map, dot11.device.last_probed_ssid_record) or other dot11.client.* structures. .netxml <wireless-client><client-mac> is also included.

Hidden/placeholder SSIDs like <hidden> or null strings are ignored.

Script 2: monitor_probes.py
What it does

Continuously monitors the directory for changes in Kismet files, extracts probe SSIDs (the networks clients are searching for), and updates an aggregate log only for unknown items:

An event is filtered out if SSID is known or client MAC is known from known_entities.json (or the legacy file).

The aggregate log unknown_probes.json is updated after every change (atomic write).

Aggregate JSON structure

Top-level keys are source filenames. Each source has a ssids map; each SSID has clients, and each client keeps a comma-separated timestamps string plus counters and first/last seen times.

This puts the source dataset first to avoid repeating the same source metadata across many entries.

CLI
python3 monitor_probes.py
  [--poll 10]
  [--entities known_entities.json]
  [--known-ssids known_ssids.json]
  [--agg-file unknown_probes.json]
  [--include-known]
  [--verbose]


--poll (default 10): seconds between scans.

--entities (default known_entities.json): path to the SSID+client reference file written by the build script.

--known-ssids (fallback): legacy reference file (list or dict with optional ssids/clients).

--agg-file (default unknown_probes.json): aggregate output file (continuously updated).

--include-known: debug switch — do not filter out known SSIDs/clients (useful to verify extraction).

--verbose: print diagnostics (counts per source, fallback paths used).

Usage
# 1) Build the known entities first
python3 build_known_ssids.py

# 2) Start the monitor (updates unknown_probes.json continuously)
python3 monitor_probes.py --poll 5

# (Optional) Debug: show/log everything, even if known
python3 monitor_probes.py --include-known --verbose --poll 5

Filtering logic

An extracted (SSID, client MAC) probe event is ignored if:

SSID ∈ known_entities["ssids"] OR

client MAC ∈ known_entities["clients"]

Otherwise, it is aggregated under:

unknown_probes.json
  -> <source file>
      -> ssids[<SSID>]
         -> clients[<client MAC>]
            -> timestamps (comma-separated)

Notes & Behaviors

Change detection: Files are re-scanned when their size or mtime changes. New files are picked up automatically.

Timestamps: Local time formatted as ISO-8601 with offset (e.g., 2025-08-16T11:02:41+0200).

Atomic writes: Output JSON files are written via a temp file and then replaced to avoid partial writes.

Hidden SSIDs: <hidden> and null/zero strings are skipped.

Heuristics: Scripts tolerate multiple Kismet schema variants and JSON field layouts (maps vs. lists).

Performance: For very large DBs, increase --poll to reduce churn, or run the build script on a subset first.

Troubleshooting

known_entities.json is empty or tiny

Ensure your captures contain beaconing APs (for SSIDs) and/or client activity (for known clients).

Run with --verbose to see which tables/columns are inspected.

unknown_probes.json stays empty

There may be no probe requests in the data, or everything is filtered as “known”.

Run the monitor with --include-known --verbose to confirm extraction is working.

Verify that known_entities.json isn’t overly broad (e.g., contains everything).

Different Kismet versions/exports

The scripts already scan multiple JSON paths and, if needed, all tables for JSON blobs.

If you see unexpected key names in your DB/XML, capture a small anonymized sample and adapt the field accessors similarly.

Happy probing!
