# claims_to_ccsr

A single-file Python pipeline that ingests healthcare claims and clinical data in the formats external partners actually send — generic tabular files, CMS extract layouts, X12 837 EDI, HL7 v2, and FHIR R4 — and emits standardized, Tableau-ready analytic tables with full lineage, clinical-vocabulary grouping, and versioned-reference governance.

One run can mix all five format families. Format detection is automatic per file.

The pipeline also supports jurisdiction-scale deployment across many independently-submitting facilities: per-file fault isolation (one bad file can't crash a batch), facility attribution, cross-run duplicate detection, a per-file data-quality gate, and optional claim-id pseudonymization. See [Jurisdiction-scale deployment](#jurisdiction-scale-deployment) below. All of it is additive — a single-facility run with none of the new flags behaves exactly as before.

## What it produces

All outputs are written to `--output-dir` and are designed to relate on `claim_id + source_file` in Tableau. Every diagnosis/DRG/procedure/pharmacy output additionally carries a `facility_id` column (see below).

| Output | Grain | Content |
|---|---|---|
| `diagnoses_ccsr.csv` | One row per (claim, ICD-10-CM code, CCSR category) | Normalized diagnosis codes mapped to AHRQ CCSR condition categories, with `is_default_category` flag and optional code-level CC/MCC severity flag |
| `claims_drg.csv` | One row per claim | Billed MS-DRG passed through from the source, enriched with CMS title, MDC, MED/SURG type, and relative weight when `--msdrg-table` is supplied |
| `procedures.csv` | One row per claim-line CPT/HCPCS code | Code type classification (CPT-I/II/III, PLA, HCPCS-II), modifiers where the source carries them, RBCS category/subcategory/family, and Medicaid H/T service-category supplement with per-row category source |
| `pharmacy_ndc.csv` | One row per NDC instance | NDCs normalized to 11-digit (5-4-2) billing format where derivable, named against the FDA NDC directory; ambiguous 10-digit unhyphenated NDCs retained raw and flagged |
| `unmapped_codes.csv` | One row per distinct code | Normalized ICD-10-CM codes absent from the CCSR crosswalk, with frequencies |
| `reference_manifest.json` | Per run | SHA-256, size, mtime, filename version hint, and row count for every reference file, plus a self-hash of the pipeline code |
| `run_summary.txt` | Per run | Per-file format detection, row counts, match rates, exclusion and deduplication accounting |
| `ingestion_report.json` / `.csv` | One row per input file | Facility, format, rows in, codes out, per-file match rate, and status (`ok`, `empty`, `quarantined`, `quarantined_low_yield`, `skipped_duplicate`, `no_diagnosis_columns`) — the operational artifact for a multi-facility deployment |
| `quarantine_report.json` | One row per failed file | Filename, facility, and the specific exception for any file that failed to parse — written only if at least one file failed |
| `quarantined_low_yield_codes.csv` | One row per excluded code | Diagnosis rows from any file below the per-file match-rate floor, excluded from `diagnoses_ccsr.csv` — written only with `--quarantine-low-yield` and at least one low-yield file |

## Supported input formats

Auto-detected per file (tabular layout can be forced with `--format`):

- **Tabular (CSV/TSV/TXT/XLSX)** in three layouts: *wide* (DX1..DX25 style columns, including CCLF/RIF naming conventions), *long* (one row per code), and *delimited* (codes packed in one field, e.g. `"E11.9; I10, Z79.4"`). Column detection is heuristic and requires no per-source configuration.
- **X12 837 EDI** — detected by the leading ISA header regardless of extension. Diagnoses from HI composites (ABK/ABF/ABJ/APR/ABN), billed DRG from `HI*DR`, claim ID from CLM01, procedures from SV1/SV2 service lines with modifiers, NDCs from `LIN*N4` drug identification. ICD-9 qualifiers are skipped and counted.
- **HL7 v2** — detected by the leading MSH segment. Diagnoses from DG1, billed DRG from the DRG segment, claim ID from PV1-19 → PID-18 → MSH-10 fallback, procedures from PR1/FT1, NDCs from RXA/RXE/RXO/RXD. Multiple messages per file and MLLP framing are handled.
- **FHIR R4** — single resource, Bundle, or bulk-export NDJSON. Diagnoses from `Condition.code` and `Claim`/`ExplanationOfBenefit` `diagnosis[]`; a run-wide Condition index resolves `diagnosisReference` entries even when Conditions live in a different file (bulk-export layout). Billed DRG from `diagnosis[].packageCode`; procedures from `item.productOrService` and Procedure resources; NDCs from pharmacy EOB items and Medication* resources. Billed diagnoses are distinguished from clinical assertions (problem-list Conditions).

## Usage

```bash
python claims_to_ccsr.py \
    --ccsr DXCCSR_v2026-1.csv \
    --msdrg-table cms_table5.csv \
    --cc-mcc cc_mcc_list.csv \
    --rbcs rbcs.csv \
    --ndc-dir fda_ndc_package.csv \
    --medicaid-xwalk hca_ht_crosswalk.csv \
    --output-dir ./out \
    claims_q1.csv encounters_837.txt adt_feed.hl7 bulk_export.ndjson
```

Only `--ccsr` is required; every other reference is optional and its enrichment simply doesn't appear if omitted.

### Governance pinning

```bash
# Approve a reference set: run once, review, save the manifest
python claims_to_ccsr.py --ccsr ... --output-dir ./approved_run inputs...
cp approved_run/reference_manifest.json approved_manifest.json

# Production runs verify against the pin
python claims_to_ccsr.py --ccsr ... \
    --pin-manifest approved_manifest.json --strict-references \
    --output-dir ./out inputs...
```

With `--pin-manifest`, any drift between the current reference files and the approved manifest is recorded as a warning; `--strict-references` refuses to run on any mismatch. This catches the case where a reference file is republished under an unchanged filename. `--hash-inputs` additionally fingerprints input claim files (off by default; inputs can be large).

## Jurisdiction-scale deployment

The flags below extend the pipeline from a single-batch tool to one that many independently-submitting facilities can feed into, over independent schedules and unreliable connectivity, without one bad file or one resubmission corrupting the run. None of them change behavior for a caller not using them.

| Flag | Purpose |
|---|---|
| `--facility-map FILE` | CSV with `file_pattern` (regex) and `facility_id` columns. Every output row is tagged with the matching facility. A file matching no pattern is tagged `UNMAPPED:<filename>`, never dropped or left blank. |
| `--ledger FILE` | Cross-run JSON record of every input file's SHA-256. A file already recorded is skipped (not silently dropped — counted as `skipped_duplicate`) unless `--allow-reingest` is given. Guards against a resubmitted claims file being double-counted. |
| `--allow-reingest` | With `--ledger`: process files anyway even if already recorded. |
| `--min-match-rate PCT` | Per-file (not just aggregate) CCSR match-rate floor, default `5.0`. A file below this is flagged low-yield in the ingestion report — a strong signal of a wrong export or reference-vintage mismatch that an aggregate match rate would otherwise mask. |
| `--quarantine-low-yield` | Exclude a low-yield file's diagnosis rows from the merged output (written to `quarantined_low_yield_codes.csv` instead) rather than only flagging it. |
| `--hash-claim-ids` | Replace `claim_id` with a namespaced one-way hash in every output. Off by default. |
| `--hmac-key-env NAME` | Environment variable holding a secret for keyed HMAC claim-id hashing (used only with `--hash-claim-ids`). Without it, hashing falls back to an unkeyed SHA-256 and prints a warning that this is demo-grade, not production pseudonymization. |
| `--max-reference-age-days N` | Warn (don't block) if a reference file's detected version hint is older than N days (default `400`). Distinct from `--pin-manifest`'s drift check: this catches a reference that's unchanged because nobody updated it, not one that changed unexpectedly. |

```bash
python claims_to_ccsr.py \
    --ccsr DXCCSR_v2026-1.csv \
    --facility-map village_clinics.csv \
    --ledger ingestion_ledger.json \
    --min-match-rate 5 --quarantine-low-yield \
    --hash-claim-ids --hmac-key-env CCSR_HMAC_KEY \
    --output-dir ./out \
    clinicA_q1.csv clinicB_q1.hl7 clinicC_q1_bulk.ndjson
```

**Fault isolation.** Every input file's processing — whichever format it's dispatched to — is wrapped individually. A malformed or truncated file (a dropped connection mid-upload, a corrupted export) is caught, its specific error recorded to `quarantine_report.json`, and the run continues for every other file. Previously an unhandled exception in one file would crash the entire batch, discarding output already computed for every other facility.

**What this doesn't do.** No distributed or streaming compute — every file still loads fully into memory, which is fine at current volumes and a scaling decision to revisit once actually demonstrated as a bottleneck, not before. No transport-level integrity check — this pipeline only sees a file once it's already on local disk, so a truncated-but-syntactically-valid transfer is caught only indirectly, via the low-match-rate gate. `--facility-map` is a hand-maintained flat file, not a governed identity registry. `--hmac-key-env` reads a secret from an environment variable, which is a real improvement over no key at all but not a managed secret store.

## Reference files

| Reference | Source | Flag |
|---|---|---|
| CCSR crosswalk | AHRQ HCUP DXCCSR (e.g. `DXCCSR_v2026-1.csv`) | `--ccsr` (required) |
| MS-DRG table | CMS IPPS Final Rule Table 5 (CSV/XLSX) | `--msdrg-table` |
| CC/MCC list | CMS MS-DRG Definitions Manual CC/MCC file | `--cc-mcc` |
| RBCS | CMS Restructured BETOS Classification System (data.cms.gov) | `--rbcs` |
| FDA NDC directory | FDA NDC directory package file | `--ndc-dir` |
| Medicaid H/T crosswalk | Agency-supplied service-category crosswalk (code + category columns) | `--medicaid-xwalk` |

All references are versioned annually (or continuously, for the NDC directory). Which version is in force is a governance decision — the manifest and pinning mechanism exist to make that decision enforceable.

## Design commitments and limitations

- **Not a DRG grouper.** MS-DRGs cannot be derived from diagnosis codes alone; assignment requires the CMS GROUPER over the full claim. The pipeline passes through billed DRGs and enriches them — it never computes them. The code-level CC/MCC flag reflects a code's *potential* severity status, not its realized contribution to the billed DRG.
- **Ambiguous NDCs are never guessed.** A 10-digit NDC supplied without hyphens cannot be safely converted to 11-digit billing format (the padded segment is unknowable), so these are retained raw with `ndc_status = ambiguous-10` for source-system follow-up.
- **RBCS under-covers Medicaid H/T codes.** RBCS is Medicare-derived. The pipeline supplements it with the agency crosswalk and, where the crosswalk is silent, coarse defaults from the national HCPCS code-block structure — every row labeled with its `medicaid_category_source` (`crosswalk`, `range-default`, or uncategorized), and per-run counts reported so crosswalk population can be prioritized by volume.
- **Outputs are claim-level PHI.** The pipeline runs locally and transmits nothing, but outputs contain claim identifiers and diagnosis detail and belong in governed storage.
- **A file's fate never depends on another file's.** Fault isolation, per-file coverage gating, and cross-run deduplication all follow from the same premise: at multi-facility scale, a crash or a bad match rate in one submission can't be allowed to cost every other facility's already-processed data.
- **Claim-id hashing is optional and honestly labeled, not on by default.** A single facility's internal batch job may reasonably treat `claim_id` as low enough risk on its own; aggregating across many facilities changes that calculus, which is what `--hash-claim-ids` is for. When used without a real key, the pipeline says so — an unkeyed hash is demo-grade, not the keyed pseudonymization a production jurisdiction-wide deployment needs.
- **Validated on synthetic fixtures to date**, covering every format family; not yet validated against production-scale data.

## Data quality accounting

Every run writes an audit trail: per-file format detection with columns/segments used, CCSR match rate, unmapped-code frequencies, ICD-9 and non-conforming values excluded (with counts), same-claim duplicates removed, ambiguous/invalid NDC counts, and H/T categorization coverage. Every output row retains lineage back to its source file, the specific column or message segment the code came from, and the submitting facility.

At single-facility scale, `run_summary.txt`'s aggregate accounting is usually enough. At multi-facility scale, `ingestion_report.json`/`.csv` extends the same accounting to a per-file, per-facility grain — which specific file was quarantined, low-yield, or a duplicate, and why — since an aggregate match rate or exclusion count can mask exactly which submission needs follow-up.

## Dependencies

- Python 3.10+
- `pandas` (required)
- `openpyxl` (only if reading `.xlsx` inputs)

## Repository layout

```
claims_to_ccsr.py        # the pipeline (single file, stdlib + pandas)
README.md                # this file
docs/                    # design notes, module highlights
tests/                   # synthetic fixtures per format family (recommended)
```
