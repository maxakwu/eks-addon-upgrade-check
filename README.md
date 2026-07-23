# eks-addon-upgrade-check

Pre-upgrade compatibility check for AWS EKS managed add-ons. Grades a proposed add-on upgrade against multiple signals — API compatibility, config schema diff, current install health, IAM posture, upstream changelog scan, and a curated known-issue registry — then emits a Markdown/JSON report with a CI-gate exit code.

Single-file bash script. Runs on macOS default bash (3.2.57), Amazon Linux 2, Amazon Linux 2023, and base Debian/Ubuntu. No installs beyond `aws` CLI v2, `jq`, `curl`, and `sha256sum` or `shasum`.

## Quickstart

```bash
chmod +x eks-addon-upgrade-check.sh

# Discover and grade every installed add-on on a cluster
./eks-addon-upgrade-check.sh --cluster <name> --region <region>

# Single add-on against a specific target version
./eks-addon-upgrade-check.sh --cluster <name> --region <region> \
  --addon aws-ebs-csi-driver --target-version v1.49.0-eksbuild.1

# CI gate — non-zero exit on any BREAKING finding
./eks-addon-upgrade-check.sh --cluster <name> --min-grade breaking \
  --format json --json-out report.json
```

Exit codes: `0` clean, `1` FYI, `2` SOFT, `3` BREAKING, `10` usage error, `11` AWS API error, `12` manifest error, `13` partial (some add-on couldn't be fully analyzed).

## What the rules manifest does — and running standalone

The rules manifest (`rules-manifest.json`) is the script's grading rulebook. It carries the per-add-on adapter dispatch, changelog source URLs, skip-level version policies, POSIX-ERE breaking-change regexes, PR-label allowlists, and the curated known-issue registry (S7) where known upgrade pitfalls live.

**You do not need to ship `rules-manifest.json` alongside the script.** A copy is baked into the script as a heredoc fallback. When you run the script:

- If `rules-manifest.json` sits next to the script (or `--rules-manifest-url` points to one), it's used and the manifest source is reported as `external`.
- If not, the embedded copy is used, one `[warn] external manifest unreachable — using embedded fallback` line prints, and the source is reported as `embedded`.

Both paths produce byte-identical reports (same SHA, same findings) as long as the two copies are in sync.

Adopters who want to pin a specific manifest version for CI can pass `--rules-sha <sha256>` — the script dies with `E_MANIFEST` (12) on mismatch, so a manifest drift never silently changes grading behavior.

## Grading signals

| Signal | Source | Confidence tier |
|---|---|---|
| S1 | `describe-addon-versions` compatibility with target K8s | `api-derived` |
| S1b | Skip-level jump vs manifest `skipLevelPolicy.maxMinorJump` | `api-derived` |
| S2 | `describe-addon-configuration` JSON-schema diff (current ↔ target) | `schema-derived` |
| S3 | Current install health (`describe-addon.status`) | `api-derived` |
| S4 | `serviceAccountRoleArn` presence when IAM is required | `api-derived` |
| S5 | Upstream changelog / release-notes regex scan | `single-signal-heuristic` |
| S7 | Curated known-issue registry (per-version-range entries) | `curated` |

## Further reading

- [DEPLOY.md](DEPLOY.md) — install, IAM, invocations, CI examples, operational notes
- [TESTING.md](TESTING.md) — three test tiers (offline smoke, fixtures, real cluster), regression harness, common issues
