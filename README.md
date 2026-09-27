# whynobid-rtb-diff

> Forensic OpenRTB diagnostics toolkit that flattens, tabulates, and ranks bid-request fields to instantly pinpoint why one cohort bids and another returns 204 / no-bid.
Act as a Senior Software Engineer. I need you to deeply analyze a GitHub repository (I will provide the link or code) and identify the most valuable, clever, or reusable piece of code inside it. This could be a core algorithm, a helpful utility function, an automation script, or an optimized process. Based on your analysis, extract this code and format it perfectly for a GitHub Gist publication. Please provide the output strictly adhering to the following structure:
>
> Gist Description: [Write a clear, concise, and professional description of what the extracted code does. Include exactly 1 relevant emoji character that fits the context of the script].
>
> Filename: [Provide the appropriate filename, including the correct file extension].
>
> Code: [Insert the extracted and refactored code here. Ensure the code is as detailed as possible, properly indented, and includes clean, professional comments explaining the core logic.]

[![Gist](https://img.shields.io/badge/gist.github-version_of_this_repository-DCDCDC?style=for-the-badge&logo=github)](https://gist.github.com/OstinUA/738e2ca3475be0373b51b17aa374b0b2)

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache--2.0-blue?style=for-the-badge)](LICENSE)
[![Python: 3.8+](https://img.shields.io/badge/Python-3.8%2B-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)
[![Version: v2](https://img.shields.io/badge/Version-v2.0.0-green?style=for-the-badge)](compare_bids_v2.py)
[![No Dependencies](https://img.shields.io/badge/Dependencies-Zero-lightgrey?style=for-the-badge)](compare_bids_v2.py)
[![RTB: OpenRTB 2.5/2.6](https://img.shields.io/badge/OpenRTB-2.5%2F2.6-orange?style=for-the-badge)](https://www.iab.com/guidelines/openrtb/)

**whynobid-rtb-diff** ingests thousands of raw OpenRTB bid requests (`*.json`, `*.txt`, `*.jsonl`, `*.ndjson`) recursively, normalizes every nested object into deterministic dot-path columns, and produces two audit-ready CSVs plus a console report that ranks fields by normalized information gain. Perfect for AdOps, DSP integration, and supply-path forensics when you need to answer: *why did Group A monetize and Group B not?*

## Table of Contents

- [Features](#features)
- [Tech Stack & Architecture](#tech-stack--architecture)
  - [Core Stack](#core-stack)
  - [Project Structure](#project-structure)
  - [Key Design Decisions](#key-design-decisions)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
- [Testing](#testing)
- [Deployment](#deployment)
- [Usage](#usage)
  - [Basic Usage](#basic-usage)
  - [Interpreting Output](#interpreting-output)
  - [Advanced Usage](#advanced-usage)
- [Configuration](#configuration)
- [License](#license)
- [Support the Project](#support-the-project)

## Features

- **Zero-dependency Python** — only stdlib (`argparse`, `json`, `csv`, `glob`, `math`, `collections`), runs anywhere Python 3.8+ exists.
- **Recursive multi-format ingestion** — walks directory tree for `*.json`, `*.txt`, `*.jsonl`, `*.ndjson`; handles:
  - Single JSON object per file
  - JSON array of objects per file
  - NDJSON / JSONL (one object per line, tolerant to blank lines and malformed lines)
- **Lossless flattening** — nested dicts become `dot.path` keys, lists of scalars joined by `;`, lists of dicts indexed as `prefix[0].field`; ensures no signal is missed.
- **Dual CSV exports**:
  - `bid_comparison.csv` — curated, stable schema of 30+ RTB-critical fields (floor, banner, device, app, regs, schain)
  - `bid_flat_all.csv` — wide export with *every* flattened field, perfect for pivot tables / BI / Athena
- **Smart grouping engine**:
  - `--groups "substr=label,substr=label"` filename-substring tagging (e.g. `Bid_request=bids,dsp_bid_request=nobid`)
  - `--group-by-folder` uses containing subfolder as label
  - Fallback `(ungrouped)` for unmatched files
- **Information-theoretic ranking (v2)** — computes normalized information gain `IG / H(group)` ∈ [0,1] via Shannon entropy over value distributions. `1.00` means field value alone perfectly predicts group. Fixes naive intersection logic from v1.
- **Perfect-separation detection** — ★ marks fields where each observed value occurs in exactly one group (pairwise disjoint, not just empty global intersection).
- **Numeric forensics** — auto-detects mostly-numeric curated columns (≥60% parseable as float) and prints per-group `n / min / median / max` (critical for `bidfloor`, `tmax`, `at`).
- **PII-aware redaction (`--redact`)** — masks `ifa`, `idfa`, `aaid`, `dpidsha1`, `dpidmd5`, `didsha1`, `didmd5`, `macsha1`, `macmd5`, `ip`, `ipv6`, `buyeruid`, `geo.lat`, `geo.lon`, `user.id` in wide export.
- **Robust MISSING sentinel** — distinguishes absent field from present-but-empty (`""`) to avoid false positives during separation analysis; rendered as blank in CSV.
- **Derived RTB intelligence**:
  - `imp_media` — infers `banner|video|native|audio` from first impression
  - `imp_count` — length of `imp[]` array (flags multi-imp anomalies)
  - `schain_last_asi` — extracts `source.ext.schain.nodes[-1].asi` or `source.schain.nodes[-1].asi`
- **Production-hardened I/O** — UTF-8 with graceful skip on `OSError`/`UnicodeDecodeError`, detailed `[skip]` logging, deterministic sorted output.
- **v1 backward compatibility** — `compare_bids_v1.py` retains simple perfect-separation logic for lightweight pipelines.

> [!NOTE]
> **v2 is the recommended entry point.** v1 is kept for auditability and minimal environments where `math` entropy is undesirable. Both scripts produce identical CSV schemas for curated fields (v2 adds `imp_count`).

## Tech Stack & Architecture

### Core Stack

| Layer | Technology | Rationale |
|-------|------------|-----------|
| Language | Python 3.8+ | Ubiquitous in AdOps / Data Eng, no runtime compilation |
| Parsing | `json`, custom NDJSON fallback | Tolerates real-world log dumps (mixed arrays + lines) |
| Flattening | Recursive `flatten()` | Converts arbitrary OpenRTB extensions (`ext`) to queryable columns |
| Analytics | `math.log2` entropy, `Counter` | Information gain without numpy/pandas dependency |
| Output | `csv.DictWriter` | Excel / Sheets / DuckDB compatible, streaming write |
| CLI | `argparse` | POSIX-compliant flags, auto-generated `--help` |

> [!TIP]
> No pandas, no numpy, no external deps. This is intentional — the tool must run on locked-down bastion hosts, Airflow workers, and SSP log servers without `pip install`.

### Project Structure

<details>
<summary><strong>📁 Full repository tree (click to expand)</strong></summary>

```
whynobid-rtb-diff/
├── LICENSE                 # Apache-2.0
├── README.md               # This file
├── compare_bids_v1.py      # v1: curated + wide CSV + simple perfect-separation report
└── compare_bids_v2.py      # v2: + JSONL/NDJSON, + entropy ranking, + PII redact, + numeric summaries, + MISSING sentinel
```

Each script is self-contained and executable (`chmod +x`). No package layout required.

</details>

```
whynobid-rtb-diff/
├── compare_bids_v2.py  # <-- Use this
├── compare_bids_v1.py  # legacy
└── LICENSE
```

### Key Design Decisions

**1. Flatten-first, curate-second**  
Raw OpenRTB is deeply nested and extension-heavy (`imp.ext`, `device.ext`, `app.publisher.ext`, `source.ext.schain`). Flattening to `dot.path` guarantees no field is invisible. Curated map (`CURATED`) then provides stable column names for dashboards.

**2. MISSING sentinel vs empty string**  
`MISSING = object()` ensures `{"battr": []}` (empty) ≠ absent. Critical for DSPs where omission vs empty array has different auction semantics. Stringification only happens at ranking time.

**3. Entropy ranking over naive diff**  
v1 checked if value sets had zero global overlap (`∩_groups == ∅`). This fails with 3+ groups where A∩B≠∅ but A∩B∩C=∅. v2 computes `H(Group) - H(Group|Field)` normalized by `H(Group)`, and perfect separation is `len(Counter(group))==1` per value. This correctly handles overlapping but predictive fields.

**4. First-impression curation**  
Most app traffic is single-imp. `imp0.*` flatten keeps curated CSV narrow and deterministic while wide CSV retains `imp[0]`, `imp[1]` etc. `imp_count` flags anomalies.

**5. PII redaction at flatten layer**  
Redaction happens after flattening but before CSV write, matching leaf name or suffix. Preserves schema (column exists, value = `<redacted>`) so downstream parsers don't break.

<details>
<summary><strong>🧭 Data flow & system design (Mermaid)</strong></summary>

```mermaid
flowchart TD
    A[Input Folder\n*.json/*.txt/*.jsonl/*.ndjson\nrecursive glob] --> B[load_files\nparse_requests\n- single object\n- array\n- NDJSON lines]
    B --> C{Per Request}
    C --> D[flatten()\n dot.path + [i] indexing\n scalar lists joined by ;]
    D --> E[Derived Fields\n imp_media()\n schain_last_asi()\n imp_count]
    E --> F[lookup table\n wide + imp0 + derived]
    F --> G1[Curated Row\n CURATED map\n MISSING sentinel]
    F --> G2[Wide Row\n every flattened key]
    G1 --> H[Group Assignment\n --groups substr=label\n or --group-by-folder]
    G2 --> H
    H --> I[CSV Writers\n bid_comparison.csv\n bid_flat_all.csv\n cell() renders MISSING as blank]
    H --> J[Analytics]
    J --> J1[numeric_summary\n >=60% float parseable\n min/median/max per group]
    J --> J2[separation_ranking\n entropy IG\n normalized 0..1\n perfect disjoint check]
    J1 --> K[Console Report]
    J2 --> K
    I --> L[Output Dir]
```

**Pipeline stages explained:**

1. **Discovery**: `glob` with `**` recursive, deduped via `set()`, sorted for determinism.
2. **Parsing**: Tries `json.loads(full_text)` first (object or array). On `JSONDecodeError`, falls back to NDJSON line-by-line tolerant parse.
3. **Normalization**: `flatten()` is pure recursion, no mutation. Scalar list detection via `all(not isinstance(x, (dict,list)))`.
4. **Enrichment**: `imp_media` scans banner/video/native/audio presence; `schain_last_asi` handles both `source.ext.schain` and legacy `source.schain`.
5. **Redaction (optional)**: In-place masking based on `PII_LEAF` exact leaf match and `PII_SUFFIX` suffix match.
6. **Export**: Two CSVs written with `extrasaction="ignore"` to stay resilient to schema drift.
7. **Ranking**: Entropy computed per value bucket, weighted by `n_v / n`.

</details>

## Getting Started

### Prerequisites

- **Python**: 3.8 or newer (`python3 --version`). No virtualenv required but recommended.
- **OS**: Linux / macOS / Windows (WSL). Uses only POSIX `os.path` and `glob`.
- **Storage**: ~2x input size for CSV outputs (wide CSV can be 5-10x wider than curated).
- **Optional**: `flake8`, `mypy`, `black` for linting if you extend the tool.

> [!IMPORTANT]
> Input files must be UTF-8. Files with `UTF-16` BOM or gzip compression must be decompressed/decoded first. The loader will `[skip]` them with a diagnostic rather than crash.

### Installation

```bash
# 1. Clone the repository
git clone https://github.com/adops-tool/whynobid-rtb-diff.git
cd whynobid-rtb-diff

# 2. Verify Python version
python3 --version  # should be >=3.8

# 3. (Optional) Create isolated env
python3 -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# 4. Make scripts executable
chmod +x compare_bids_v1.py compare_bids_v2.py

# 5. Quick smoke test
python3 compare_bids_v2.py --help
```

Expected `--help` output:

```
usage: compare_bids.py [-h] [--groups GROUPS] [--group-by-folder] [--redact]
                       [--top TOP] [--outdir OUTDIR]
                       folder
```

<details>
<summary><strong>🔧 Troubleshooting & Alternative Installs</strong></summary>

**No README found? Use v2 directly:**
```bash
curl -O https://raw.githubusercontent.com/adops-tool/whynobid-rtb-diff/main/compare_bids_v2.py
python3 ./compare_bids_v2.py /path/to/logs --groups "bid=ok,204=nobid"
```

**Permission denied on macOS:**
```bash
xattr -d com.apple.quarantine compare_bids_v2.py
```

**Large log folders (100k+ files) cause `Argument list too long`:**
The tool uses `glob` internally, not shell expansion — always pass the folder, not `*.json`:
```bash
# Correct
python3 compare_bids_v2.py /data/rtb_logs/2024-09-27/

# Wrong (shell expands)
python3 compare_bids_v2.py /data/rtb_logs/2024-09-27/*.json
```

**Docker one-liner (no local Python):**
```bash
docker run --rm -v $(pwd):/work -v /path/to/logs:/logs:ro python:3.11-slim \
  python3 /work/compare_bids_v2.py /logs --outdir /work/out --redact
```

**Building from source (editable):**
```bash
# No build step needed — single file module
# If you want to package:
pip install build
python -m build --wheel  # after adding pyproject.toml
```

</details>

## Testing

This repository is intentionally dependency-free; tests are manual + static analysis.

```bash
# 1. Syntax check
python3 -m py_compile compare_bids_v1.py compare_bids_v2.py

# 2. Lint (if flake8 installed)
pip install flake8
flake8 compare_bids_v2.py --max-line-length=120 --ignore=E203,W503

# 3. Type check (optional)
pip install mypy
mypy compare_bids_v2.py --ignore-missing-imports --check-untyped-defs

# 4. Unit-style smoke test with synthetic data
mkdir -p /tmp/rtb_test/bids /tmp/rtb_test/nobid
cat > /tmp/rtb_test/bids/a.json <<'JSON'
{"id":"req1","at":2,"imp":[{"banner":{"w":320,"h":50},"bidfloor":0.5}],"app":{"bundle":"com.example"},"device":{"os":"android"}}
JSON
cat > /tmp/rtb_test/nobid/b.json <<'JSON'
{"id":"req2","at":2,"imp":[{"banner":{"w":320,"h":50},"bidfloor":5.0}],"app":{"bundle":"com.example"},"device":{"os":"ios"}}
JSON
python3 compare_bids_v2.py /tmp/rtb_test --group-by-folder --top 10 --outdir /tmp/rtb_out
cat /tmp/rtb_out/bid_comparison.csv
# Expected: bidfloor 0.5 vs 5.0 shows high separation score

# 5. Test NDJSON and redaction
cat > /tmp/rtb_test/mixed.jsonl <<'JSONL'
{"id":"r1","imp":[{"bidfloor":1}],"device":{"ifa":"abc-123","ip":"1.2.3.4"}}
{"id":"r2","imp":[{"bidfloor":2}],"device":{"ifa":"def-456","ip":"5.6.7.8"}}
JSONL
python3 compare_bids_v2.py /tmp/rtb_test --redact --outdir /tmp/rtb_out
grep redacted /tmp/rtb_out/bid_flat_all.csv && echo "redaction OK"

# 6. Full integration with your own logs
python3 compare_bids_v2.py /path/to/production_logs --groups "Bid_request=bids,dsp_bid_request=nobid" --top 30 --outdir ./out
```

> [!NOTE]
> No `pytest` suite is bundled to keep zero deps. If you integrate into CI, wrap the smoke test above in a shell script and assert exit code `0` and existence of `bid_comparison.csv` + `bid_flat_all.csv`.

<details>
<summary><strong>🧪 Edge-case matrix to validate</strong></summary>

| Case | Input | Expected Behavior |
|------|-------|-------------------|
| Empty file | `0 bytes` | `[skip] no JSON object found` |
| Malformed JSON | `{"id":}` | Skipped, counted in console but not CSV |
| JSON array | `[{...},{...}]` | Expands to `file#0`, `file#1` |
| NDJSON with blank lines | `{"id":1}\n\n{"id":2}` | 2 rows, blank ignored |
| Multi-imp | `imp:[{banner..},{video..}]` | `imp_count=2`, `imp_media=banner` (first) |
| Missing schain | `source:{}` | `schain_last=""` blank, not crash |
| PII present + --redact | `device.ip` | `<redacted>` in wide CSV, curated untouched |
| All fields constant | 100 identical files | Ranking prints `(no curated field shows separation)` |
| 3+ groups | `a=1,b=2,c=3` | Entropy ranking handles N groups, perfect check is pairwise disjoint |

</details>

## Deployment

### Local / Ad-hoc

```bash
python3 compare_bids_v2.py /var/log/rtb/ --groups "200=bids,204=nobid" --outdir ./out_$(date +%F)
ls -lh ./out_*/bid_*.csv
```

### Docker / Compose

```dockerfile
# Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY compare_bids_v2.py .
ENTRYPOINT ["python3", "compare_bids_v2.py"]
```

```yaml
# docker-compose.yml
version: "3.9"
services:
  whynobid:
    build: .
    volumes:
      - ./logs:/logs:ro
      - ./out:/out
    command: "/logs --groups Bid_request=bids,dsp_bid_request=nobid --redact --top 30 --outdir /out"
```

```bash
docker compose up --build
```

### CI/CD Integration

```yaml
# .github/workflows/rtb-diff.yml
name: rtb-diff
on: [push]
jobs:
  diff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: python3 compare_bids_v2.py tests/fixtures --group-by-folder --outdir out
      - uses: actions/upload-artifact@v4
        with:
          name: bid-csvs
          path: out/*.csv
```

> [!WARNING]
> **Wide CSV can be large.** A single OpenRTB request with deep `ext` can flatten to 200+ columns. For 100k requests, `bid_flat_all.csv` may exceed 2GB. Use `--outdir` on a volume with sufficient space and consider piping curated CSV only to BI if needed.

> [!CAUTION]
> Never commit raw `bid_flat_all.csv` containing IFA/IP/geo to Git. Always run with `--redact` for any artifact that leaves your VPC, and add `*.csv` to `.gitignore`.

## Usage

### Basic Usage

```bash
# 1. Just tabulate everything in a folder (single group)
python3 compare_bids_v2.py /path/to/folder

# Output:
# Loaded 1243 request(s) from /path/to/folder
# Wrote ./bid_comparison.csv
# Wrote ./bid_flat_all.csv
# Only one group present. Pass --groups ...

# 2. Compare two cohorts by filename substring (most common)
python3 compare_bids_v2.py /path/to/folder \
  --groups "Bid_request=bids,dsp_bid_request=nobid" \
  --outdir ./out

# 3. Group by subfolder name (e.g., /logs/bids/*.json vs /logs/204/*.json)
python3 compare_bids_v2.py /path/to/folder --group-by-folder

# 4. Full forensics with PII redaction and top 50 ranked fields
python3 compare_bids_v2.py /path/to/folder \
  --groups "bids=bids,204=nobid" \
  --redact \
  --top 50 \
  --outdir ./forensics
```

**Python as a library (import flatten):**

```python
from compare_bids_v2 import flatten, build_rows, schain_last_asi

# Flatten any OpenRTB dict
req = {
  "id": "abc",
  "imp": [{"banner": {"w": 300, "h": 250}, "bidfloor": 1.2}],
  "device": {"os": "ios", "ifa": "XXXX"},
  "source": {"ext": {"schain": {"nodes": [{"asi": "example.com"}]}}}
}

flat = flatten(req)
print(flat["imp[0].banner.w"])  # 300
print(schain_last_asi(req))     # example.com

# Build curated + wide rows (with redaction)
curated, wide = build_rows(req, redact_pii=True)
print(curated)  # {'request_id': 'abc', 'bidfloor': 1.2, ...}
print(wide["device.ifa"])  # <redacted>
```

### Interpreting Output

**`bid_comparison.csv` (curated):**

```csv
file,group,request_id,auction_type,tmax,imp_count,imp_media,bidfloor,floor_cur,instl,secure,ban_w,ban_h,api,battr,pos,displaymanager,dm_ver,tagid,app_id,app_bundle,app_name,pub_id,os,osv,make,model,devicetype,conn,lmt,dnt,coppa,gdpr,schain_last
a.json,bids,req-1,2,300,1,banner,0.5,USD,0,1,320,50,3;5,,0,,,...,com.app.test,...,android,13,Samsung,SM-G998,1,2,0,0,,0,example.com
b.json,nobid,req-2,2,300,1,banner,5.0,USD,0,1,320,50,3;5,,0,,,...,com.app.test,...,ios,16,Apple,iPhone14,1,6,0,0,,1,other.com
```

**Console report (v2):**

```
Numeric fields — min / median / max by group:

  bidfloor:
        bids         n=120  min=0.1000 median=0.5000 max=1.2000
        nobid        n=80   min=3.0000 median=5.0000 max=12.0000

Field separation ranking — how strongly each curated field predicts the group
(1.00 = the field's value alone tells the groups apart; 0.00 = no signal)

  [1.00] bidfloor  ★ perfectly separates
        bids         0.5 (60/120), 0.1 (30/120), 1.2 (30/120)
        nobid        5.0 (50/80), 3.0 (30/80)
  [0.92] os
        bids         android (110/120), ios (10/120)
        nobid        ios (75/80), android (5/80)
  [0.45] schain_last
        bids         example.com (100/120), direct (20/120)
        nobid        other.com (80/80)
```

> [!TIP]
> Start with `[1.00]` and ★ fields — they are deterministic blockers. Then examine high-but-not-perfect scores (0.7-0.99) for probabilistic filters like `os`, `api`, `battr`, `gdpr`.

### Advanced Usage

<details>
<summary><strong>🎯 Advanced grouping strategies</strong></summary>

```bash
# 3-way comparison: bids vs timeout vs no-bid
python3 compare_bids_v2.py /logs \
  --groups "200=bids,timeout=timeout,204=nobid" \
  --top 40

# Group by publisher bundle (pre-process with symlink folders)
mkdir -p /tmp/by_bundle
for f in /logs/*.json; do
  bundle=$(jq -r '.app.bundle // "unknown"' "$f")
  mkdir -p "/tmp/by_bundle/$bundle"
  ln -sf "$f" "/tmp/by_bundle/$bundle/"
done
python3 compare_bids_v2.py /tmp/by_bundle --group-by-folder --outdir ./bundle_report
```

</details>

<details>
<summary><strong>🔒 PII redaction deep dive</strong></summary>

When `--redact` is set, the wide CSV masks:

- **Leaf exact match**: `ifa`, `idfa`, `aaid`, `dpidsha1`, `dpidmd5`, `didsha1`, `didmd5`, `macsha1`, `macmd5`, `ip`, `ipv6`, `buyeruid`
- **Suffix match**: paths ending with `geo.lat`, `geo.lon`, `user.id`

Curated CSV is **not** redacted (it contains no PII by design). If you add custom curated fields that are PII, extend `PII_LEAF`/`PII_SUFFIX`.

```python
# Extending redaction
PII_LEAF.add("my_custom_id")
PII_SUFFIX = (*PII_SUFFIX, "ext.my_pii")
```

</details>

<details>
<summary><strong>🧩 Extending curated fields</strong></summary>

Edit `CURATED` dict in `compare_bids_v2.py`:

```python
CURATED = {
    # ... existing
    "device.geo.country": "country",
    "device.geo.city": "city",
    "app.cat": "app_cat",
    "imp0.banner.mimes": "mimes",
    "imp0.video.minduration": "vid_min_dur",
    "imp0.video.maxduration": "vid_max_dur",
    "imp0.pmp.private_auction": "pmp_private",
}
```

Then re-run. Wide CSV already contains these fields automatically; curated just gives them stable column names.

**Custom formatter example:**

```python
def build_rows(req, redact_pii=False):
    # ... original
    lookup["imp0.banner.api_str"] = "|".join(map(str, imp0.get("banner", {}).get("api", [])))
    # Add to CURATED: "imp0.banner.api_str": "api_str"
```

</details>

<details>
<summary><strong>⚠️ Edge cases & gotchas</strong></summary>

- **Empty `imp[]`**: `imp_count=0`, `imp_media=""`, all `imp0.*` become `MISSING` → blank in CSV, counted as `«absent»` in ranking.
- **Scalar list vs dict list**: `["a","b"]` → `"a;b"`. `[{"id":1},{"id":2}]` → `field[0].id=1`, `field[1].id=2`. Wide CSV keeps `[0]`, curated uses only `[0]` via `imp0`.
- **Duplicate files**: `glob` deduped via `set()`, sorted. Symlinks followed by OS.
- **Large NDJSON**: Entire file read into memory (`fh.read()`). For >500MB NDJSON, split with `split -l 10000`.
- **Floating precision**: `bidfloor` parsed as float for summary, but ranking uses stringified bucket to avoid `0.5 != "0.5"` issues.
- **Timezone**: No time parsing; add `device.ext` timestamp to `CURATED` if needed.

</details>

## Configuration

All configuration is via CLI flags. No external config file required.

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `folder` | positional | — | Folder to recursively scan for bid files |
| `--groups` | string | `""` | Comma-separated `substring=label` rules. First match wins. Example: `"Bid_request=bids,dsp=nobid"` |
| `--group-by-folder` | bool | `false` | Use immediate parent folder name as group instead of `--groups` |
| `--redact` | bool | `false` | Mask PII in `bid_flat_all.csv` (see PII tables) |
| `--top` | int | `30` | How many ranked separating fields to print to console |
| `--outdir` | path | `"."` | Directory to write CSVs. Created if missing. |

> [!IMPORTANT]
> `--groups` matching is **substring** and **case-sensitive**. Use lowercase substrings if your files are lowercased, or use `--group-by-folder` for deterministic grouping.

<details>
<summary><strong>📋 Exhaustive configuration tables</strong></summary>

**Curated Field Map (`CURATED` dict: dot-path → CSV column):**

| Dot-path | CSV Column | Derived? | Type | Notes |
|----------|------------|----------|------|-------|
| `id` | `request_id` | No | string | OpenRTB request ID |
| `at` | `auction_type` | No | int | 1=first price, 2=second price |
| `tmax` | `tmax` | No | int | Max timeout ms |
| `imp_count` | `imp_count` | Yes | int | `len(imp)` |
| `imp0.media` | `imp_media` | Yes | string | banner/video/native/audio inferred |
| `imp0.bidfloor` | `bidfloor` | No | float | Floor CPM |
| `imp0.bidfloorcur` | `floor_cur` | No | string | Currency, usually USD |
| `imp0.instl` | `instl` | No | int | Interstitial flag |
| `imp0.secure` | `secure` | No | int | 1=secure impression |
| `imp0.banner.w` | `ban_w` | No | int | Banner width |
| `imp0.banner.h` | `ban_h` | No | int | Banner height |
| `imp0.banner.api` | `api` | No | string | API frameworks (joined ;) |
| `imp0.banner.battr` | `battr` | No | string | Blocked creative attributes |
| `imp0.banner.pos` | `pos` | No | int | Ad position |
| `imp0.displaymanager` | `displaymanager` | No | string | SDK name |
| `imp0.displaymanagerver` | `dm_ver` | No | string | SDK version |
| `imp0.tagid` | `tagid` | No | string | Placement ID |
| `app.id` | `app_id` | No | string | App ID |
| `app.bundle` | `app_bundle` | No | string | Bundle / package name |
| `app.name` | `app_name` | No | string | App name |
| `app.publisher.id` | `pub_id` | No | string | Publisher ID |
| `device.os` | `os` | No | string | OS (android/ios) |
| `device.osv` | `osv` | No | string | OS version |
| `device.make` | `make` | No | string | Device make |
| `device.model` | `model` | No | string | Model |
| `device.devicetype` | `devicetype` | No | int | 1=mobile, 4=phone, 5=tablet |
| `device.connectiontype` | `conn` | No | int | 2=wifi, 3=cell, etc. |
| `device.lmt` | `lmt` | No | int | Limit ad tracking |
| `device.dnt` | `dnt` | No | int | Do not track |
| `regs.coppa` | `coppa` | No | int | COPPA flag |
| `regs.ext.gdpr` | `gdpr` | No | int | GDPR flag |
| `schain_last_asi` | `schain_last` | Yes | string | Last schain node ASI |

**PII Redaction Rules:**

| Set | Match Type | Values |
|-----|------------|--------|
| `PII_LEAF` | Exact leaf segment after last `.` | `ifa`, `idfa`, `aaid`, `dpidsha1`, `dpidmd5`, `didsha1`, `didmd5`, `macsha1`, `macmd5`, `ip`, `ipv6`, `buyeruid` |
| `PII_SUFFIX` | `str.endswith()` on full dot-path | `geo.lat`, `geo.lon`, `user.id` |

**Environment Variables (optional, not required):**

No env vars are read by default. If wrapping in shell:

```bash
export RTB_LOG_DIR=/var/log/rtb
export RTB_OUT_DIR=./out
export RTB_GROUPS="200=bids,204=nobid"
python3 compare_bids_v2.py "$RTB_LOG_DIR" --groups "$RTB_GROUPS" --outdir "$RTB_OUT_DIR" --redact
```

**Default `.env` template (if you create wrapper):**

```ini
# .env.example — not auto-loaded, for your wrapper script
RTB_FOLDER=/data/rtb_logs
RTB_GROUPS=Bid_request=bids,dsp_bid_request=nobid
RTB_TOP=30
RTB_OUTDIR=./out
RTB_REDACT=true
```

</details>

## License

Licensed under the **Apache License 2.0** — see [LICENSE](LICENSE) for full text.

```
Copyright 2024 OstinUA

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

> [!NOTE]
> Apache-2.0 permits commercial use, modification, distribution, patent use, and private use. You must include license and copyright notice and state changes.

## Support the Project

[![Patreon](https://img.shields.io/badge/Patreon-OstinFCT-f96854?style=flat-square&logo=patreon)](https://www.patreon.com/OstinFCT)
[![Ko-fi](https://img.shields.io/badge/Ko--fi-fctostin-29abe0?style=flat-square&logo=ko-fi)](https://ko-fi.com/fctostin)
[![Boosty](https://img.shields.io/badge/Boosty-Support-f15f2c?style=flat-square)](https://boosty.to/ostinfct)
[![YouTube](https://img.shields.io/badge/YouTube-FCT--Ostin-red?style=flat-square&logo=youtube)](https://www.youtube.com/@FCT-Ostin)
[![Telegram](https://img.shields.io/badge/Telegram-FCTostin-2ca5e0?style=flat-square&logo=telegram)](https://t.me/FCTostin)

If you find this tool useful, consider leaving a star on GitHub or supporting the author directly.
