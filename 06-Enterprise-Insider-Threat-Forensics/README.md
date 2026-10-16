# Enterprise Insider Threat Investigation

A full forensic investigation of a suspected insider-threat data-exfiltration incident, carried across three pieces of evidence (a USB drive, a corporate system image, and a RAM capture) from acquisition through chain-of-custody documentation, Autopsy analysis, and memory forensics — ending in the raw carving of AES-256 key material from a corrupted memory dump.

> **Scope.** This is a graduate DFIR coursework investigation against an instructor-provided scenario, not a real incident. The case number (`2026-05-12-CORP`), the suspect (`Nicholas Anderson` / "Nick"), the victim org (`Global Tech Corp`), and the HR/IT personnel are the lab's own fictional labels, not real persons or organizations. Hashes, serial numbers, and drive labels in the PDFs are the scenario's own values, reproduced as generated — they are not redacted, because there is nothing real to redact.

## Tools

| Tool | Version | Purpose |
|---|---|---|
| Autopsy | 4.21.0 | File-system analysis, entropy scoring, raw data carving, ingest tracking |
| FTK Imager | 4.2.0.13 | Forensically sound bit-stream imaging and MD5 hash validation |
| Volatility | 2.6 | Structured volatile-memory analysis (process listing, kernel block identification) |
| Bulk Extractor | 1.5.2 | Raw cryptographic-key carving from the corrupted memory dump |

## Artifacts in this directory

| Path | Contents |
|---|---|
| [`forensic-report.pdf`](./forensic-report.pdf) | Final forensic report — executive summary, evidence inventory, hash verification, methodology, and findings |
| [`working-notes.pdf`](./working-notes.pdf) | Investigator working notes — timestamped acquisition and analysis timeline across all three items |
| [`evidence-control-log.pdf`](./evidence-control-log.pdf) | Evidence Control Log — intake/verification record with per-item MD5 hashes |
| [`chain-of-custody-form.pdf`](./chain-of-custody-form.pdf) | Chain-of-Custody tracking form — released-by/received-by handoffs and final-disposition authority |

## Evidence items

| Item | Original evidence | Image | Size | Acquisition MD5 | Autopsy validation |
|---|---|---|---|---|---|
| 001 | Lexar USB flash drive (S/N `AACWGT047W9DWD7V`) | `image5.dd.001` | 3824 MB | `51d3851fe4181ceb98fff16e11cc6e71` | Match |
| 002 | VMware virtual IDE hard drive | `winimage.001` | 15360 MB | `d39c651a4f522d2e13bd0429e2de9e3f` | Match |
| 003 | Windows 10 memory capture | `Windows 10 x64-31acf216.vmem` | 2048 MB | `33201c5f2cfd81bbcafbbbd371122431` | Match |

All three acquisition hashes matched Autopsy's ingest-time hashes, establishing an unbroken integrity chain from collection to analysis. Full handoff record in [`chain-of-custody-form.pdf`](./chain-of-custody-form.pdf) and [`evidence-control-log.pdf`](./evidence-control-log.pdf).

## Methodology

Full timeline: [`working-notes.pdf`](./working-notes.pdf). Full write-up: [`forensic-report.pdf`](./forensic-report.pdf).

1. **Acquisition under chain of custody.** Each item was connected through a hardware write-blocker, imaged, hashed (MD5), and logged to the Evidence Control Log and Chain-of-Custody form before any analysis, per NIST evidence-handling guidance.
2. **USB analysis (Item 001).** `image5.dd.001` was ingested into Autopsy. Entropy scoring flagged `stuff.txt` (entropy `7.999982` — near-perfect randomness) as likely ciphertext; `VeraCrypt Setup 1.26.7.exe` in the root established the encryption method. Recovered PDFs (an internal email, a resignation letter) established the suspect's identity and motive.
3. **System-image analysis (Item 002).** `winimage.001` was added as a second data source. Manual navigation plus ingest surfaced the installed VeraCrypt, a second 10 MB high-entropy container (`1.txt`), the `MountedDevices` registry hive, and recent-`.lnk` artifacts.
4. **Memory forensics (Item 003).** The `.vmem` capture was profiled in Volatility and process-listed (`pslist`/`pstree`). Structured key extraction failed due to memory smearing introduced during acquisition, so the investigation pivoted to raw carving with Bulk Extractor.

## Findings

**Encryption tooling and staged containers.** VeraCrypt was installed on both the USB drive and the corporate system (`C:\Program Files\VeraCrypt`, `C:\Users\Nick\Documents\VeraCrypt`). Two high-entropy containers disguised as text — `stuff.txt` on the USB and a second file sized at exactly 10 MB, `1.txt`, on the system drive — are consistent with encrypted VeraCrypt volumes holding staged data.

**Volumes were actively mounted and browsed.** The `SYSTEM` hive's `MountedDevices` key proves VeraCrypt volumes were mounted as drives `K:` and `L:`. Windows `.lnk` artifacts (`Local Disk (K).lnk`, `Local Disk (L).lnk`, `1.lnk`) place the user actively navigating *inside* the encrypted volumes — `1.lnk` points to a file opened within the `L:` volume.

**Supporting tooling.** A 7-Zip installer (`7z2201-x64.exe`) in the suspect's Documents is a plausible staging/compression tool; `FTK Imager Lite` on the system traces to IT's pre-handover imaging and is the likely source of the memory capture.

**AES-256 key material carved from corrupted RAM.** Volatility's structured extraction was defeated by memory smearing introduced when IT acquired the capture. Raw cryptographic carving with Bulk Extractor bypassed the corruption and recovered several **candidate** AES-256 master keys. The plaintext VeraCrypt password was **not** recovered; the carved keys are decryption candidates to be tested against the `1.txt` and `stuff.txt` containers (e.g. with `vcdecrypt`), not a confirmed decryption.

## Assessment

The artifacts are mutually corroborating: motive and off-hours database access (from recovered correspondence), the means (installed VeraCrypt), the act (two encrypted containers plus mounted-volume and `.lnk` evidence of the user working inside them), and partial key recovery from memory despite acquisition-induced corruption. The investigation's notable technical result is methodological — when structured memory analysis failed, raw carving still produced usable cryptographic candidates, demonstrating that a corrupted dump is not automatically a dead end. The honest limit is that recovery stopped at candidate keys: no container was actually decrypted within the exercise.

## Recommendations

- **Preserve memory captures correctly.** The smearing that defeated structured analysis was introduced during acquisition. Capture volatile memory with a validated tool before the system is otherwise touched, and hash immediately — the difference here between structured extraction and last-resort carving.
- **Alert on unsanctioned encryption tooling.** Installation of VeraCrypt (or equivalents) on an endpoint, and mounting of unrecognized encrypted volumes, should be an alertable event, not something only found post-incident.
- **Baseline off-hours data access.** The originating signal was database access outside working hours. Pair access logging with conditional-access policy so off-hours access to sensitive stores is an exception that fires, not routine noise.
- **Watch for staging behavior.** Large fixed-size high-entropy files and on-host archivers (7-Zip) in user profiles are staging indicators worth correlating with removable-media and volume-mount events.

## Limitations

- Coursework investigation against an instructor-provided scenario; all identities and organizations are fictional lab labels.
- Key recovery reached **candidate** AES-256 keys only — no container was decrypted, and the plaintext password was not recovered.
- Structured memory analysis was partially defeated by acquisition-induced smearing; findings from Item 003 rest on raw carving rather than clean structured extraction.
- The full artifact set lives in the four PDFs above; this README is a summary, not a substitute for the report.
