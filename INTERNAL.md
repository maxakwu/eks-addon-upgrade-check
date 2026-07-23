# Internal notes — eks-addon-upgrade-check

Maintainer-facing documentation. Not intended for public consumption. Covers design references, sync/release mechanics, portability policy, and the intentional-deferral backlog.

For public-facing usage, see [README.md](README.md).

## Design references

Full design for this standalone script and its companion API feature — internal artifacts:

- [01-eks-addon-upgrade-check-script.md](01-eks-addon-upgrade-check-script.md) — full design for the shell script (grading model, signal taxonomy S1–S8, downgrade rule, rendering, extensibility, testing plan). Section 6 is the canonical grading-rules reference.
- [02-eks-addon-upgrade-check-api-feature.md](02-eks-addon-upgrade-check-api-feature.md) — companion design for a first-class EKS API surface (`AnalyzeAddonUpgrade`, per-cluster enablement, EventBridge on grade transitions, Console tab, IAM additions).

## Refresh the embedded manifest heredoc

The script embeds a copy of `rules-manifest.json` as a bash heredoc so it can run standalone. When you edit `rules-manifest.json`, regenerate the embedded copy before release:

```bash
# From the project root, on any platform with python3:
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

Verify both paths still produce the same SHA and same grading:

```bash
# External path
./eks-addon-upgrade-check.sh --cluster <c> --region <r> --refresh 2>&1 | head -2
# Note the SHA and (source=external)

mv rules-manifest.json rules-manifest.json.bak
# Embedded path
./eks-addon-upgrade-check.sh --cluster <c> --region <r> --refresh 2>&1 | head -3
# Note the SHA and (source=embedded) — SHA must match the external run
mv rules-manifest.json.bak rules-manifest.json
```

## Portability policy (contributors)

The script targets bash 3.2 with GNU or BSD userland interchangeably, so the same file runs on macOS, AL2, AL2023, and base Debian/Ubuntu. If you're adding code, follow these four rules — otherwise the script will silently mis-behave on one platform. Full detail lives in [DEPLOY.md §7.1](DEPLOY.md).

1. **No bash 4+ features.** No `mapfile` / `readarray`. No `declare -A`. No `${var,,}` / `${var^^}`. No `local -n` namerefs. No `**` globstar.
2. **Guard empty-array expansions** under `set -u`: use `${arr[@]+"${arr[@]}"}`, not plain `"${arr[@]}"`.
3. **Portable coreutils flags only.** `stat -c` (GNU) vs `stat -f` (BSD) require both branches. `date -d`, `readlink -f` are GNU-only. `sed -i` semantics differ across platforms.
4. **POSIX ERE only in the manifest.** No `(?i)`, `\s`, `\d`, `\w`, `(?:...)`. Test regexes on BSD `grep -E` before merging.

The preflight check in `preflight()` will fail early with `bash >= 3.2 required` if run under an older shell.

## Follow-ups (intentional deferrals)

Items known-not-present in v0, tracked here so future contributors know they're gaps by design:

- **LICENSE file** — needed before public distribution. Apache 2.0 recommended (matches AWS samples convention).
- **Script header** — copyright / license / SPDX one-liner at the top of `eks-addon-upgrade-check.sh`.
- **`.gitignore`** — cache dir (`~/.cache/eks-addon-check` and platform equivalents), `report.json`, `metrics.json`, editor swap files, `.DS_Store`.
- **CI smoke workflow** — a GitHub Actions job that runs `bash -n eks-addon-upgrade-check.sh` and `jq -e . rules-manifest.json` on every push. ~15 lines of YAML.
- **Manifest `signature.digest`** — currently `PLACEHOLDER-COMPUTE-AND-BAKE-INTO-SCRIPT`. Either compute + bake into the script as a pinned default `RULES_SHA_EXPECTED`, or remove the `signature` field entirely.
- **Automated test suite** — TESTING.md documents a manual playbook. Fixture-based `bats` tests would let the manifest be refreshed weekly without regressions. Recorded API responses via `moto` or equivalent, one fixture per grading-rule row from §6.2 of the design doc.
- **Curated `knownIssues` seed** — the launch manifest ships with one entry (EBS CSI 1.49 pre-flight). Others worth seeding: VPC CNI prefix-delegation limit hits, CoreDNS readiness probe timing changes across specific versions, kube-proxy iptables → nftables mode transitions.

## Release checklist

Before tagging a release:

1. Refresh the embedded heredoc (see above).
2. Bump `SCRIPT_VERSION` in `eks-addon-upgrade-check.sh`.
3. Bump `manifestVersion` in `rules-manifest.json` (date-based: `YYYY-MM-DD.N`).
4. Run all three test tiers from [TESTING.md](TESTING.md).
5. Verify external + embedded manifest paths produce identical SHAs and identical grading against a scratch cluster.
6. `git tag vX.Y.Z && git push --tags`.
