# Module highlights — `claims_to_ccsr.py`

A guided tour of the module's architecture and the design decisions worth knowing before reading or extending the code. The file is a single ~2,450-line module organized into clearly delimited sections; this document follows that order. This revision folds in the jurisdiction-scale deployment layer added this turn — fault isolation, facility attribution, per-file quality gating, cross-run idempotency, reference staleness, and optional claim-id pseudonymization — on top of the original single-batch architecture described below, which is otherwise unchanged.

## Architecture at a glance

```
inputs (any mix)                      parsers                     outputs
─────────────────                     ───────                     ───────
CSV / TSV / XLSX  ──► read_any ─► detect_format ─► to_long ─┐
X12 837 EDI       ──► is_x12_file  ─► parse_x12_file  ──────┤    diagnoses_ccsr.csv
HL7 v2            ──► is_hl7_file  ─► parse_hl7_file  ──────┼──► claims_drg.csv
FHIR R4           ──► is_fhir_file ─► parse_fhir_file ──────┘    procedures.csv
                                                                  pharmacy_ndc.csv
reference files ─► loaders ─► fingerprint ─► manifest ─► pin     unmapped_codes.csv
                                    │                             reference_manifest.json
                                    └─► staleness check            run_summary.txt

per-file, before any parsing:                                    NEW, this turn:
  resolve_facility_id() ─► fingerprint ─► check_ledger()          ingestion_report.json/.csv
    │ (skip if already ingested, unless --allow-reingest)         quarantine_report.json
    ▼                                                             quarantined_low_yield_codes.csv
  [try: parse] ── exception ──► quarantined (run continues)
    │ success
    ▼
  tag_facility() on every output frame ─► per-file coverage gate
    (--min-match-rate; --quarantine-low-yield excludes, doesn't crash)
```

Every parser still converges on the same four tidy frames — diagnoses, claim-level DRGs, claim-line procedures, NDC lines — so downstream grouping, enrichment, and quality accounting stay format-agnostic. Each row carries `source_file`, the specific column or segment it came from, and now `facility_id`. What's new sits entirely *around* that convergence point — one file's fate (quarantined, low-yield, duplicate, or clean) no longer depends on, or endangers, any other file's.

## Section-by-section

**Column-name heuristics (config).** Diagnosis, procedure, NDC, claim-ID, and DRG columns are found by case-insensitive regex families covering common conventions — `DX1`, `DIAG_CD_2`, `ICD10_1`, CCLF/RIF names, `PROD_SRVC_ID` — so new tabular sources in these families require no new code. `_matches_any()` is the single dispatch point, which keeps the heuristics auditable in one place.

**Code normalization.** `normalize_code()` canonicalizes ICD-10-CM (case, dots, whitespace) and rejects non-conforming values rather than passing them through; rejects are counted, not silently dropped. The same discipline repeats for procedures (`normalize_procedure_code` + `classify_procedure_code` distinguishing CPT-I/II/III, PLA, and HCPCS-II by shape) and DRGs (`normalize_drg`).

**Reference versioning and pinning.** `file_fingerprint()` records SHA-256, size, and mtime; `detect_version_hint()` scrapes version strings like `v2026-1` or `FY2026` from filenames; `build_reference_manifest()` assembles these plus row counts and a self-hash of the pipeline source into `reference_manifest.json`. `verify_against_pin()` compares the live manifest to a governance-approved one — warning on drift, or refusing to run under `--strict-references`. The design insight: approving a reference set becomes signing off a JSON file, and enforcement is automatic thereafter. The self-hash means code changes are drift too.

`check_reference_staleness()` (new) checks a distinct failure mode from drift: a reference file that hasn't changed because nobody has gotten around to updating it. `parse_hint_to_asof_date()` turns the same version hint used above into an approximate as-of date and compares it against `--max-reference-age-days` (default 400) — a genuinely unparseable hint is reported as "cannot assess," never silently treated as current. This only matters once a pipeline is running unattended, on a schedule, over a long operational lifetime — a single analyst's one-off run has no use for it, which is why it's a warning, not a gate.

**Facility identity (new).** A filename alone isn't a stable identity once many sites submit into one pipeline — the same PM/EHR vendor's default export name recurs across unrelated facilities, and one facility's own naming convention can change over time. `load_facility_map()` reads a flat `file_pattern` (regex) → `facility_id` CSV; `resolve_facility_id()` tries each pattern against the filename, first match wins. An unmatched file is never dropped or left blank — it's tagged `UNMAPPED:<filename>`, so it still shows up, attributably, in every downstream report even with no facility map configured at all. Every extracted frame gets this tag via a single `tag_facility()` call at the point each frame is produced, rather than threading a new parameter through every format parser.

**Ingestion ledger (new).** `load_ingestion_ledger()` / `save_ingestion_ledger()` maintain a cross-run JSON record of every input file's SHA-256 — reusing the same fingerprinting built for reference pinning, applied here to inputs across runs instead of references within one. `check_ledger()` is a pure content-address lookup: a resubmitted file under a new filename with identical bytes is still caught; a genuinely corrected file under the same filename is correctly treated as new. A file already on record is skipped (counted as `skipped_duplicate`, never silently dropped) unless `--allow-reingest` is given. The case where *every* input turns out to be a duplicate is deliberately **not** treated as an error — it exits 0 with a plain "nothing new to process" message, since a scheduled jurisdiction-wide job's orchestration needs to distinguish a clean no-op from a real failure.

**NDC handling.** `normalize_ndc()` returns a `(value, status)` pair rather than a bare value — `ok`, `ambiguous-10`, or `invalid`. Hyphenated 10-digit NDCs are padded to 11-digit 5-4-2 deterministically; unhyphenated 10-digit values are *kept raw and flagged*, because the missing segment position is unknowable. Encoding "we don't know" as a first-class status, instead of guessing or dropping, is the pattern the whole module follows.

**Claim-id pseudonymization (new, optional).** Off by default — `claim_id` still passes through raw exactly as before, unchanged for anyone not using the new flag. `--hash-claim-ids` applies `hash_claim_id()` to every output carrying a claim identifier via one `apply_claim_id_hash()` call per output frame, right before each `to_csv()`. The design follows the same keyed/unkeyed honesty split as the downstream analytics platform's MBI/FIN hashing: a real secret supplied through `--hmac-key-env` produces a keyed HMAC; absent that, the pipeline still runs, but falls back to an unkeyed SHA-256 with a printed warning that this is demo-grade, not the server-held-key pseudonymization a real jurisdiction-wide deployment requires. The rationale for making this optional at all: a single covered entity's internal batch job may reasonably treat `claim_id` as low-risk on its own, but aggregating claim-level data across an entire jurisdiction's facilities is a different risk profile, and the flag exists specifically for that transition, not as a default assumption about risk that may not hold for every deployment.

**Medicaid H/T supplement.** RBCS is Medicare-derived and under-covers H/T codes. The section layers three sources with explicit provenance: agency crosswalk → `medicaid_range_default()` coarse HCPCS block defaults → uncategorized. Each procedure row records its `medicaid_category_source`, and the run summary reports coverage counts so crosswalk population can be prioritized by observed volume rather than guesswork.

**CMS reference loaders.** `load_ccsr`, `load_msdrg_table`, `load_cc_mcc`, `load_rbcs`, and `load_ndc_directory` each tolerate the real-world variability of their published file (column naming, CSV vs. XLSX) and normalize to a fixed internal schema, so the rest of the module never touches raw reference layouts.

**Tabular format detection and reshaping.** `detect_format()` classifies each file as wide, long, or delimited (`looks_packed()` samples a column for multi-code fields); `to_long()` reshapes all three into the tidy grain, preserving dx position and source column. Deduplication of same-claim repeat codes happens here, with counts surfaced in the summary.

**HL7 v2 parser.** Hand-rolled segment parsing (no HL7 library dependency): DG1 for diagnoses with coding-system checks (ICD-9 DG1s skipped and counted), DRG segment pass-through, RXA/RXE/RXO/RXD for pharmacy, PR1/FT1 for procedures, and a claim-ID fallback chain PV1-19 → PID-18 → MSH-10. MLLP framing bytes and multi-message files are handled.

**X12 837 parser.** Detects ISA headers regardless of file extension, tokenizes on the declared element/segment separators, reads HI composites by qualifier (ABK principal, ABF other, ABJ admitting, APR reason-for-visit, ABN external cause — with ICD-9 qualifiers BK/BF/BJ/PR/BN skipped and counted), `HI*DR` for billed DRG, CLM01 for claim ID, SV1/SV2 service lines with modifiers, and `LIN*N4` for drug NDCs.

**FHIR R4 parser.** The most structurally interesting section. `_iter_fhir_resources()` walks single resources, Bundles, and NDJSON uniformly. Because bulk exports split resources across files, `main()` does a first pass over all FHIR inputs to build a *run-wide* Condition index, so a `Claim.diagnosis[].diagnosisReference` in one file resolves against a Condition in another. Conditions consumed by a claim reference are suppressed from standalone emission (tracked by object identity) to avoid double counting, and problem-list Conditions are distinguished from billed diagnoses — a source-of-truth distinction, not just a parsing detail. ICD-9-only codings and unresolvable references are skipped and counted.

**Main.** Orchestration, now in two passes before the format-specific parsing even starts. First, a resolution pass over every input: `resolve_facility_id()` tags a facility, and — if `--ledger` is given — `file_fingerprint()` plus `check_ledger()` decide whether this exact file has already been ingested in a prior run; duplicates are skipped (recorded, not silently dropped) before any parsing work, including the FHIR pre-pass, is spent on them. What survives that pass becomes `active_inputs`, and only those go on to the FHIR Condition-index pre-pass and per-file dispatch described above.

The per-file dispatch loop itself is now wrapped in a `try/except` per file — the single most consequential change this turn. Previously, one facility's malformed or truncated file (a dropped rural connection mid-upload, a corrupted export) would raise an unhandled exception and crash the run for every other file already processed, since nothing was written until the loop finished. A caught exception is now recorded to a `quarantined` list with its specific error and type, printed, and the loop continues — every other facility's data ships regardless of what happened to one bad file.

After concatenation and the CCSR join, a **per-file** coverage gate runs (`--min-match-rate`, default 5%) — a groupby on `source_file` distinct from the aggregate match-rate accounting, because a single low-yield facility file can drag in near-zero matches that an aggregate percentage would mask entirely. `--quarantine-low-yield` excludes a flagged file's rows from the merged output and writes them to `quarantined_low_yield_codes.csv` instead of silently blending them in.

Everything from both passes — facility, format, row/code counts, match rate, quarantine/duplicate status — accumulates into a structured `file_reports` list (not just the pre-existing printed `detections` strings, which are kept for the human-readable summary) and is written to `ingestion_report.json`/`.csv`; genuine parse failures get their own `quarantine_report.json`. The ledger is updated at the end for every file actually processed this run — deliberately excluding files quarantined for a real parse error, so a fixed-and-resubmitted file is retried automatically rather than permanently treated as "already seen." Conditional outputs remain principled exactly as before: `claims_drg.csv` only appears if a DRG was actually found, and the summary notes when an optional reference was supplied but no matching codes existed in any input.

## Cross-cutting design principles

1. **Never guess.** Ambiguous NDCs flagged, DRGs never derived (the GROUPER caveat is documented in the module docstring), unresolvable FHIR references counted rather than fabricated.
2. **Everything is counted.** Every exclusion, dedup, skip, and coverage gap appears in `run_summary.txt` — the audit trail is a product of the run, not an afterthought. `ingestion_report.json`/`.csv` extends this from an aggregate account to a per-file, per-facility one.
3. **Provenance on every row.** Source file plus source column/segment, category-source labels, `is_default_category` and severity flags, and now `facility_id` — lineage survives to Tableau.
4. **Governance as mechanism, not memo.** Reference versioning is enforced by hash comparison, not by convention; the new staleness check extends this from "has it changed" to "is it current."
5. **Convergent tidy schema.** Four heterogeneous format families, one relational output shape.
6. **One file's fate is never another's (new).** Fault isolation, per-file coverage gating, and cross-run deduplication all follow from the same premise: at single-batch scale a crash or a bad match rate is one analyst's problem to notice and fix; at jurisdiction scale it's dozens of unrelated facilities' data riding on whether any one submission was clean. The failure unit shrank from "the run" to "the file."
7. **Optional means unchanged by default (new).** `--hash-claim-ids`, `--facility-map`, `--ledger`, and the coverage/staleness gates are all additive — none alters output for a caller not using them, and the original single-batch usage in the module docstring's first example still runs exactly as it did before this turn.

## Extension points

- New tabular naming conventions: add a regex to the pattern lists in the config section.
- New service-category logic: extend the Medicaid crosswalk file, not the code.
- New FHIR resource types: add a branch in `parse_fhir_file()`; the resource iterator and reference index are already generic.
- Measure value sets (e.g., contraceptive care): natural next reference family, pinned under the same manifest mechanism.
- New facilities: add a row to the `--facility-map` CSV — no code change, same pattern as the Medicaid crosswalk above.
- A different per-facility coverage floor: the coverage gate isn't implemented as a per-facility table yet, only a single global `--min-match-rate`; a facility known in advance to run structurally sparse would need that generalized into a per-facility override if it comes up in practice — the same shape of exception the downstream analytics platform's feature-store coverage gate already makes for one specific, legitimately-partial data source.
- A real facility master-data registry, a KMS-backed key store, and transport-level integrity checking are explicitly out of scope for this revision — noted in the module docstring's own "what this does not do" section rather than silently absent.

## What's honestly not addressed yet

Stated plainly, matching the module's own "never claim more than what's built" convention: no distributed or streaming compute (every file still loads fully into a `pandas.DataFrame`); no verification that a transfer completed intact before this pipeline ever sees the file (a truncated-but-syntactically-valid upload is caught only indirectly, via the low-match-rate gate, not via a checksum at the point of transfer); `--facility-map` is a hand-maintained flat file, not a governed identity system; and `--hmac-key-env` reads a secret from an environment variable, which is a real improvement over no key at all but not a managed secret store. Each is a scaling decision to make once actually demonstrated at that scale, not a gap to build around in advance of it.
