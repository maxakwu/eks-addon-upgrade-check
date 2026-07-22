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

The rules manifest (`rules-manifest.json`) is the script's grading rulebook. It carries the per-add-on adapter dispatch, changelog source URLs, skip-level version policies, POSIX-ERE breaking-change regexes, PR-label allowlists, and the curated known-issue registry (S7) where AWS-observed upgrade pitfalls live.

**You do not need to ship `rules-manifest.json` alongside the script.** A copy is baked into the script as a heredoc fallback. When you run the script:

- If `rules-manifest.json` sits next to the script (or `--rules-manifest-url` points to one), it's used and the manifest source is reported as `external`.
- If not, the embedded copy is used, one `[warn] external manifest unreachable — using embedded fallback` line prints, and the source is reported as `embedded`.

Both paths produce byte-identical reports (same SHA, same findings) as long as the two copies are in sync. When you edit `rules-manifest.json`, regenerate the embedded copy before release with:

```bash
# From the project root, on any platform with sha256sum OR shasum:
python3 -c "
import json, re, sys
manifest = open('rules-manifest.json').read()
script = open('eks-addon-upgrade-check.sh').read()
new = re.sub(
    r\"(cat > \\\"\\\$dest\\\" <<'EOF_RULES_MANIFEST'\\n).*?(\\nEOF_RULES_MANIFEST)\",
    lambda m: m.group(1) + manifest.rstrip('\\n') + m.group(2),
    script, count=1, flags=re.DOTALL)
open('eks-addon-upgrade-check.sh','w').write(new)
print('embedded manifest refreshed')
"
```

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

Full grading rules, tie-breaking, and downgrade rules are in [01-eks-addon-upgrade-check-script.md](01-eks-addon-upgrade-check-script.md) §6.

## Documentation index

- [DEPLOY.md](DEPLOY.md) — install, IAM, invocations, CI examples, operational notes, portability policy for contributors
- [TESTING.md](TESTING.md) — three test tiers (offline smoke, fixtures, real cluster), regression harness, common issues
- [01-eks-addon-upgrade-check-script.md](01-eks-addon-upgrade-check-script.md) — full design doc for this standalone script
- [02-eks-addon-upgrade-check-api-feature.md](02-eks-addon-upgrade-check-api-feature.md) — companion design for a first-class EKS API action (AnalyzeAddonUpgrade, per-cluster enablement, EventBridge on grade transitions)

## Portability

Contributors: this script targets bash 3.2 with GNU or BSD userland interchangeably. Before adding code, read the four portability rules in [DEPLOY.md §7.1](DEPLOY.md) — no `mapfile`, no `declare -A`, guard empty-array expansions, and POSIX ERE only in the manifest.

## Follow-ups (intentional deferrals)

Items not present in v0, called out here so they are visible to future contributors:

- **LICENSE** — repo needs one before public distribution. Apache 2.0 recommended (matches AWS samples).
- **Script header** — copyright / license / SPDX one-liner at the top of the script.
- **`.gitignore`** — cache dir, `report.json`, `metrics.json`, editor swap files, `.DS_Store`.
- **CI smoke workflow** — a GitHub Actions job that runs `bash -n` on the script and `jq -e .` on the manifest on every push.
- **Manifest `signature.digest`** — currently a placeholder. Either compute + bake into the script as a pinned default, or remove the field.
- **Automated test suite** — TESTING.md documents a manual playbook. Fixture-based `bats` tests would let the manifest be refreshed weekly without regressions.
