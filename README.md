# Dual-Jurisdiction Cloud Storage Compliance Scanner

> **Research tool for:** *Cloud Data Localisation Compliance under Russia's Federal Law 242-FZ and India's Digital Personal Data Protection Act 2023*

A Python-based compliance evaluation tool that checks cloud storage bucket configuration against clause-level requirements extracted from a DPDPA 2023 / 152-FZ legal analysis matrix. Outputs a structured report showing which clauses pass, which fail, which generate warnings, and which of the six identified inter-law conflicts are triggered by the configuration.

---

## What This Tool Is

A **static configuration auditor**. It reads a cloud storage configuration — either a real AWS S3 bucket via API or a simulated dictionary — and evaluates it against a set of compliance rules derived from two data protection laws:

- **India:** Digital Personal Data Protection Act 2023 (DPDPA)
- **Russia:** Federal Law No. 152-FZ on Personal Data + Federal Law No. 242-FZ (data localisation amendment, effective September 2015)

The tool was developed as part of a research paper comparing cloud data localisation requirements under these two laws. Its primary purpose is to demonstrate that the six legal conflicts identified in the paper are **detectable and measurable from concrete infrastructure configuration parameters**, not merely theoretical constructs.

---

## What This Tool Is NOT

| It is NOT | Why this matters |
|---|---|
| A network scanner | It does not probe, ping, or connect to any external system |
| A penetration testing tool | It performs no exploitation, fuzzing, or vulnerability scanning |
| A real-time regulatory feed | Blacklists and adequacy lists are defined as static constants in the code |
| A legal instrument | Findings are for research and informational purposes only |
| An enterprise compliance platform | It covers 9 specific checks relevant to the research — not hundreds |

You run it against **your own** configuration. It touches no infrastructure it is not explicitly pointed at.

---

## Checks Performed

| Check | Clause ID | What It Evaluates |
|---|---|---|
| Storage region — Russia primary | `242FZ-A18(5)` | Is the bucket in a Russian-territory region? |
| Storage region — DPDPA blacklist | `DPDP-S16` | Is the bucket in a country blacklisted by India? |
| Encryption at rest | `DPDP-S8(5)` | Is server-side encryption enabled? |
| Encryption standard — GOST | `152FZ-A19(3b)` | Is an FSB-certified GOST algorithm in use for PDIS? |
| Public access blocked | `DPDP-S8(5)` | Are all four public access block settings enabled? |
| Versioning enabled | `152FZ-A19(8)` | Is object versioning on for audit trail support? |
| Access logging enabled | `DPDP-S8(5) / 152FZ-A19` | Are server access logs being captured? |
| Retention / deletion policy | `DPDP-S8(7) / 152FZ-A5(7)` | Is there a lifecycle policy with expiry rules? |
| HTTPS-only enforced | `DPDP-S8(5) / 152FZ-A19` | Is HTTP access denied by bucket policy? |

---

## Conflict Detection

The tool evaluates all six inter-law conflicts identified in the research paper:

| Conflict ID | Laws in Tension | Triggered When |
|---|---|---|
| CONFLICT-01 | DPDP S.6(1) vs 152-FZ Art.9(4) | Storage serves both Indian and Russian data subjects |
| CONFLICT-02 | 242FZ-A18(5) vs DPDP-S16 | Region satisfies DPDPA but violates 242-FZ localisation |
| CONFLICT-03 | 152FZ-A19(3b) vs DPDP-S8(5) | AES encryption satisfies DPDPA but not 152-FZ GOST requirement |
| CONFLICT-04 | DPDP-S8(4) vs 152FZ-A19(1)/A19(3) | Missing controls acceptable under DPDPA but not under FSTEC audit |
| CONFLICT-05 | DPDP-S16 vs 152FZ-A12(1) | Transfer permitted by DPDPA but requires Art.12(4) basis under 152-FZ |
| CONFLICT-06 | DPDPA (absent) vs 152FZ-A16 | Russian data subjects in scope with potential ADM system downstream |

---

## Two Operating Modes

### Mode A — Real AWS S3 (Live configuration)

Connects to AWS using boto3 and fetches real bucket configuration via the S3 API. Calls:

- `get_bucket_location()` — region
- `get_bucket_encryption()` — encryption algorithm
- `get_public_access_block()` — public access settings
- `get_bucket_versioning()` — versioning status
- `get_bucket_logging()` — logging configuration
- `get_bucket_lifecycle_configuration()` — retention/expiry rules
- `get_bucket_policy()` — HTTPS enforcement policy

The API responses are mapped into the same configuration dictionary structure that Mode B uses, then the same compliance logic runs against it.

**Requirements:**
- AWS account (free tier is sufficient)
- boto3 installed (`pip install boto3`)
- AWS credentials configured (`aws configure` or environment variables)
- Permission to call the above S3 read APIs on the target bucket

**Use this when:** You want to evaluate a real bucket and produce findings based on actual infrastructure configuration. Stronger for research credibility.

---

### Mode B — Simulated Configuration (No cloud access)

Runs the same compliance checks against a hardcoded Python dictionary representing a bucket configuration. No network connection is made. No credentials are needed. The dictionary values are defined in the script and reflect realistic configurations a developer would produce.

Four built-in scenarios:

| Scenario | Region | Encryption | Logging | Versioning | Retention | Purpose |
|---|---|---|---|---|---|---|
| `mixed` | us-east-1 | AES256 | Off | Off | Off | Realistic — partial compliance, multiple failures |
| `pass` | ru-central1 | GOST R 34.12-2015 | On | On | On | Hypothetical fully-compliant dual-jurisdiction config |
| `fail` | eu-west-1 | None | Off | Off | Off | All checks fail — demonstrates every failure mode |
| `india_only` | ap-south-1 | AES256 | On | On | On | Indian users only — only DPDPA checks apply |

**Use this when:** You have no AWS access, are demonstrating the tool in a classroom or research context, or want to show specific failure patterns.

---

## Installation

```bash
# Clone the repository
git clone https://github.com/yourusername/dual-jurisdiction-compliance-scanner
cd dual-jurisdiction-compliance-scanner

# No external dependencies for Mode B
python compliance_scanner.py --mode b --scenario mixed

# For Mode A, install boto3
pip install boto3
```

---

## Usage

### Mode B — Simulated (no setup required)

```bash
# Default: mixed scenario
python compliance_scanner.py --mode b

# Specific scenario
python compliance_scanner.py --mode b --scenario mixed
python compliance_scanner.py --mode b --scenario pass
python compliance_scanner.py --mode b --scenario fail
python compliance_scanner.py --mode b --scenario india_only

# Override data subject origin
python compliance_scanner.py --mode b --scenario mixed --origin russian
python compliance_scanner.py --mode b --scenario mixed --origin indian
python compliance_scanner.py --mode b --scenario mixed --origin both

# Export findings to JSON
python compliance_scanner.py --mode b --scenario mixed --json report.json

# Disable colour output (for logging or piping)
python compliance_scanner.py --mode b --scenario mixed --no-colour
```

### Mode A — Real AWS S3

```bash
# Configure AWS credentials first
aws configure
# Enter: Access Key ID, Secret Access Key, Default region, Output format (json)

# Or set environment variables
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_DEFAULT_REGION=us-east-1

# Run against a real bucket
python compliance_scanner.py --mode a --bucket your-bucket-name

# Specify region explicitly
python compliance_scanner.py --mode a --bucket your-bucket-name --region eu-west-1

# Export to JSON
python compliance_scanner.py --mode a --bucket your-bucket-name --json report.json

# Override data subject origin (default: both)
python compliance_scanner.py --mode a --bucket your-bucket-name --origin russian
```

### Google Colab (no local setup)

```python
# Cell 1 — paste the script content (everything above def build_parser)
# then add at the end:
USE_COLOUR = False

# Cell 2 — run mixed scenario
config    = dict(SIMULATED_CONFIGS["mixed"])
checks    = run_checks(config)
conflicts = analyse_conflicts(config, checks)
summary   = render_report(config, checks, conflicts)
```

---

## Output Format

```
========================================================================
  DUAL-JURISDICTION COMPLIANCE SCAN REPORT
  DPDPA 2023  |  Federal Law 152-FZ / 242-FZ
========================================================================
  Target  : test-bucket-research-mixed
  Region  : us-east-1
  Origin  : both data subjects
  PDIS    : Yes
  Scanned : 2026-03-23 09:35:21 UTC
  Version : 1.0.0

  CLAUSE-LEVEL RESULTS
  ----------------------------------------------------------------------
  Clause ID                    Check                                Status
  ----------------------------------------------------------------------
  242FZ-A18(5)                 Storage region (Russia primary)      FAIL
                                 Region 'us-east-1' is not a Russian territory...
                                 → Deploy a separate Russian-region primary bucket...

  DPDP-S16                     Storage region (DPDPA blacklist)     PASS
  DPDP-S8(5)                   Encryption at rest                   PASS
  152FZ-A19(3b)                Encryption standard (GOST)           WARN
  ...

  CONFLICT ANALYSIS
  ----------------------------------------------------------------------
  CONFLICT-01   DPDP S.6(1) vs 152-FZ Art.9(4)     ACTIVE
  CONFLICT-02   242FZ-A18(5) vs DPDP-S16            ACTIVE
  ...

  SUMMARY
  ----------------------------------------------------------------------
  Checks passed   : 4 / 9
  Checks failed   : 4 / 9
  Warnings        : 1 / 9
  Skipped         : 0 / 9
  Conflicts active: 6 / 6

  OVERALL POSTURE: NON-COMPLIANT — Critical failures detected.
========================================================================
```

---

## JSON Export Structure

```json
{
  "scanner_version": "1.0.0",
  "scan_timestamp": "2026-03-23T09:35:21+00:00",
  "target": {
    "bucket_name": "test-bucket-research-mixed",
    "region": "us-east-1",
    "encryption_enabled": true,
    "encryption_algorithm": "AES256",
    ...
  },
  "check_results": [
    {
      "clause_id": "242FZ-A18(5)",
      "check_name": "Storage region (Russia primary)",
      "status": "FAIL",
      "detail": "Region 'us-east-1' is not a Russian territory...",
      "recommendation": "Deploy a separate Russian-region primary bucket..."
    },
    ...
  ],
  "conflict_results": [
    {
      "conflict_id": "CONFLICT-02",
      "laws": "242FZ-A18(5) vs DPDP-S16",
      "status": "ACTIVE",
      "detail": "Region 'us-east-1' satisfies DPDPA S.16 but violates 242-FZ Art.18(5)...",
      "recommendation": "Deploy a separate Russian-territory primary bucket..."
    },
    ...
  ],
  "summary": {
    "passed": 4,
    "failed": 4,
    "warned": 1,
    "skipped": 0,
    "active_conflicts": 6
  }
}
```

---

## Configuration Dictionary Reference

When writing your own simulated config or interpreting Mode A output:

| Key | Type | Values | Source in Mode A |
|---|---|---|---|
| `bucket_name` | string | Any | Provided by user |
| `region` | string | AWS region code | `get_bucket_location()` |
| `encryption_enabled` | bool | True / False | `get_bucket_encryption()` |
| `encryption_algorithm` | string | AES256, GOST, aws:kms, None | `get_bucket_encryption()` |
| `public_access_blocked` | bool | True / False | `get_public_access_block()` |
| `versioning_enabled` | bool | True / False | `get_bucket_versioning()` |
| `logging_enabled` | bool | True / False | `get_bucket_logging()` |
| `retention_policy_set` | bool | True / False | `get_bucket_lifecycle_configuration()` |
| `https_only_enforced` | bool | True / False | `get_bucket_policy()` |
| `data_subject_origin` | string | russian / indian / both | Declared by operator |
| `is_pdis` | bool | True / False | Declared by operator |

> **Note:** `data_subject_origin` and `is_pdis` cannot be auto-detected from bucket configuration. They depend on business context — who the users are and what the system does — and must be declared by the operator running the scan.

---

## Limitations

**Legal:** This tool does not constitute legal advice. Findings are for research and informational purposes only. Consult qualified legal counsel for actual compliance determinations.

**Scope:** The tool covers 9 configuration-level checks relevant to the specific research question. It does not cover all obligations under either law.

**Regulatory currency:** The DPDPA blacklist (`DPDPA_BLACKLISTED_REGIONS`) and Russian regions list (`RUSSIAN_REGIONS`) are defined as static constants. In a production system these would be fetched from live regulatory feeds. No such public API currently exists for either.

**Mode A scope:** The boto3 calls cover S3-level configuration. Application-level controls (consent management, breach notification pipelines, erasure workflows) are outside the scope of what any bucket-level scanner can assess.

**CONFLICT-06 (ADM):** Cannot be determined from bucket configuration alone. The conflict is flagged as contextually relevant whenever Russian data subjects are in scope, but whether an actual automated decision-making system consumes this data is a business-context question the tool cannot answer automatically.

---

## Academic Citation

If you use this tool in research, cite as:

```
Jadhav, Y. (2026). Dual-Jurisdiction Cloud Storage Compliance Scanner
[Software]. Research tool developed for comparative analysis of cloud
data localisation under India's DPDPA 2023 and Russia's Federal Law
152-FZ / 242-FZ. ITMO University / Parul University.
```

---

## Legal References

- Digital Personal Data Protection Act, 2023 (India) — No. 22 of 2023
- Federal Law No. 152-FZ of July 27, 2006 on Personal Data (Russia)
- Federal Law No. 242-FZ of July 21, 2014 (Data Localisation Amendment, Russia)
- Government Decree PP-1119 (UZ Classification Framework, Russia)
- FSTEC Order No. 21 (Security Controls for PDIS, Russia)

---

## Disclaimer

This tool was developed for academic research purposes. It is not a commercial product, not a certified compliance solution, and not a substitute for professional legal or security advice. The author makes no warranty as to the accuracy or completeness of the compliance checks relative to the full text of either law.
