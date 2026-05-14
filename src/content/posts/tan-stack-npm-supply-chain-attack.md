---
title: "TanStack npm Supply Chain Attack: What Happened and What Engineers Should Do"
published: 2026-05-13
updated: 2026-05-13
description: "A coordinated compromise hit @tanstack/*, @mistralai/*, @uipath/* and more - here's the technical breakdown and defensive guidance."
tags: [Security,TanStack,npm,Supply Chain Attack]
image: "../../assets/images/post-covers/tan-stack-npm-supply-chain-attack.webp"
category: 'Security'
draft: false 
---

> Cover image source: [lexica.art](https://lexica.art/prompt/6cbeeeb6-4c44-4677-bd29-480a3e4013f2)


A coordinated supply chain attack published malicious versions of over 170 npm packages and 2 PyPI packages on May 11–12, 2026, hitting organizations including TanStack, Mistral AI, UiPath, and OpenSearch. The campaign, tracked as "mini-shai-hulud" by StepSecurity and Socket Security is one of the largest registry poisoning events observed in 2026 and the first to span both npm and PyPI in a single operation.

# What Happened

SafeDep's malware detection pipeline flagged a burst of suspicious npm package publications on the night of May 11, 2026. Unlike previous supply chain attacks that targeted a single high-value package, this campaign went after entire organizational scopes. Every package under @tanstack, @squawk, @uipath, @tallyui, and other namespaces was published with malicious versions in a single coordinated push.

The same actor then expanded to PyPI on May 12, 2026, publishing malicious versions of the official Mistral AI Python SDK (`mistralai==2.4.6`) and the Guardrails AI validation framework (`guardrails-ai==0.10.1`). PyPI quarantined both projects. No legitimate `v2.4.6` tag exists in the `mistralai/client-python` repository - the attacker created a version that never existed in the upstream source.

TanStack published a postmortem explaining the initial compromise path, a chained GitHub Actions attack using the `pull_request_target` "Pwn Request" pattern, GitHub Actions cache poisoning across the fork-to-base trust boundary, and runtime memory extraction of an OIDC token from the GitHub Actions runner process. The malicious publishes were authenticated through the project's OIDC trusted-publisher binding after attacker-controlled code ran during the workflow's test/cleanup phase and posted directly to the npm registry. TanStack stated that no npm tokens were stolen and that the npm publish workflow itself was not compromised.

# Timeline

- **May 11, 2026 (nighttime UTC):** SafeDep detects suspicious npm package publications across multiple scopes. Socket flags 84 compromised TanStack npm package artifacts.
- **May 12, 2026 (~03:05 UTC):** Campaign expands to PyPI. `mistralai==2.4.6` and `guardrails-ai==0.10.1` published. Cloudflare marks `git-tanstack[.]com` as a suspected phishing site.
- **May 12, 2026:** npm removes malicious tarballs. TanStack deprecates affected versions, purges GitHub Actions cache entries, and merges hardening changes to the affected workflow.
- **Patched versions published** across affected packages (see below).

# How the Attack Worked

## The GitHub Actions Compromise Chain

TanStack's postmortem attributes the initial access to a combination of three GitHub Actions misconfiguration patterns:

1. **`pull_request_target`** - This workflow trigger makes the base repository's workflow code available to pull request authors. When combined with cache poisoning, it allows an attacker controlling a fork to influence trusted pipeline behavior.
2. **GitHub Actions cache poisoning** - Attackers injected malicious content into the shared workflow cache, which was later loaded by the base repository's CI pipeline.
3. **OIDC token memory extraction** - After attacker controlled code ran in the workflow, it extracted the GitHub Actions runner's OIDC token from process memory. That token was used to authenticate publishes directly to the npm registry via the project's OIDC trusted-publisher binding, bypassing the need for a static npm token entirely.

This is a significant escalation from traditional npm token theft. The attacker did not steal a stored npm API token, they stole a dynamically issued OIDC token that the CI system itself used to prove its identity to the registry.

## The Malicious Payload

The npm packages received an added `router_init.js` file - a ~2.3 MB heavily obfuscated file using javascript-obfuscator patterns (string-array rotation, hex-encoded identifier lookups, control-flow flattening inside `while(!![]){}` state machines, dead-code injection). This is distinct from normal minifier output from Terser, esbuild, or swc.

The file spawns a daemonized process that:

- Reads and exfiltrates `GITHUB_*` environment variables (GitHub Actions secrets, tokens, actor identity)
- Probes `http://169.254.169.254/latest/meta-data/iam/security-credentials/` (AWS metadata service)
- Probes `http://127.0.0.1:8200` (HashiCorp Vault)
- Downloads a platform-specific Bun runtime from `hxxps://github[.]com/oven-sh/bun/releases/download/bun-v1.3.13/`
- Downloads the main payload from `hxxps://git-tanstack[.]com/transformers.pyz`
- Communicates with `hxxp://filev2[.]getsession[.]org/file/` as the command-and-control server

The `package.json` was also modified to include:

```json
"optionalDependencies": {
  "@tanstack/setup": "github:tanstack/router#79ac49eedf774dd4b0cfa308722bc463cfe5885c"
}
```

That commit (`79ac49eedf774dd4b0cfa308722bc463cfe5885c`) is a root commit with no parent history that introduces only a `package.json` and bundled `tanstack_runner.js` payload. The `prepare` lifecycle hook runs `bun run tanstack_runner.js && exit 1`, executing arbitrary code during installation of the git-based dependency. Even before the package is fully installed in a project.

## PyPI Payload Difference

The PyPI packages use a different delivery mechanism. On import, a Python dropper downloads `transformers.pyz` from `hxxps://git-tanstack[.]com/transformers.pyz` and executes it with `python3`. This is the same attacker-controlled domain used in the npm campaign, confirming the same infrastructure.

# Who Is Affected


The attack hit over 170 npm packages across multiple organizations. The major affected packages include:

| Package | Downloads | Risk |
|---|---|---|
| `@tanstack/react-router` | 3M+ weekly | CI credential theft |
| `@tanstack/vue-router` | Widely used | CI credential theft |
| `@opensearch-project/opensearch` | 1.3M weekly | CI credential theft |
| `@uipath/robot` | Enterprise RPA | Credential theft |
| `@mistralai/mistralai` (npm) | SDK consumers | Credential theft |
| `mistralai` (PyPI) | Python SDK | Credential theft |
| `guardrails-ai` (PyPI) | ML validation | Credential theft |

Patched versions have been published for all affected packages. TanStack's security advisory lists specific affected and patched versions per package.

# What Engineers Should Do Now

## 1. Identify affected installations

Check your dependency tree for `router_init.js` files. Socket published the following SHA-256 for detection:

```
ab4fcadaec49c03278063dd269ea5eef82d24f2124a8e15d7b90f2fa8601266c
```

Run:

```sh
find . -name "router_init.js" -exec shasum -a 256 {} \;
```

## 2. Rotate all exposed secrets

If you installed any affected `@tanstack/*` version between May 11–12, 2026, rotate the following immediately in priority order:

1. **GitHub PATs and OIDC federation grants** - especially any trusts used for npm publishing
2. **npm tokens** - all active publishing tokens, even if TanStack claims none were stolen directly
3. **AWS credentials** - static keys and instance role tokens (the payload probes the EC2 metadata service)
4. **Vault tokens** - the payload probes localhost:8200
5. **SSH keys** - if any were present in the CI environment

## 3. Revoke GitHub Actions OIDC trust

The attack extracted OIDC tokens from GitHub Actions runner memory. If your organization uses OIDC trusted publishers for npm or other registry publishing, review and rotate those federation grants.

## 4. Pin and audit dependencies

- Audit `package.json` for unexpected `optionalDependencies` referencing git URLs, especially GitHub commits that aren't part of your normal dependency graph
- Audit your lockfile for newly added transitive dependencies you didn't expect
- Consider pinning `@tanstack/*` to specific versions rather than semver ranges during this period

## 5. Review CI logs

Check your GitHub Actions logs for any unusual behavior between May 11–12: unexpected network calls, spawning of unfamiliar processes, or access to secrets outside normal workflow steps.

## 6. Enable supply chain protections

The pnpm 11 release (May 4, 2026) introduced defaults specifically aimed at the conditions this attack exploited:

- **Minimum Release Age (24h default):** Newly published package versions are not resolved until they are at least one day old, reducing exposure during the highest-risk window immediately after publication
- **Block Exotic Subdeps (on by default):** Prevents transitive dependencies from resolving from git repositories or direct tarball URLs, narrowing one path attackers use to hide unexpected code

If you use pnpm, upgrading to v11 and keeping the defaults on significantly reduces the risk of fast-moving supply chain attacks succeeding silently.

For npm and Yarn users, similar protections exist via Socket's AI Scanner and SafeDep's Package Manager Guard.

# What Is Still Unknown

- **Initial access vector** - TanStack's postmortem describes the GitHub Actions exploitation chain but has not yet published the full root-cause analysis of how the attacker first compromised the `voicproducoes` GitHub account (the account used for the malicious commit)
- **Number of affected organizations** - neither npm nor TanStack have published estimates of how many projects consumed the malicious versions
- **Exploitation status** - I found no confirmed public evidence of active exploitation of stolen credentials as of May 13, 2026, but the credential-stealing payload was live in the registry for hours before removal, and the blast radius of OIDC token theft can extend beyond the immediate publisher
- **Full scope beyond npm/PyPI** - StepSecurity's tracking suggests the campaign also hit Packagist (PHP), though details are still emerging

# References

- [SafeDep: Mass Supply Chain Attack Hits TanStack, Mistral AI npm and PyPI Packages](https://safedep.io/mass-npm-supply-chain-attack-tanstack-mistral/) — Primary detection source, full technical analysis including PyPI expansion
- [Socket Security: TanStack npm Packages Compromised in Ongoing Mini Shai-Hulud Supply-Chain Attack](https://socket.dev/blog/tanstack-npm-packages-compromised-mini-shai-hulud-supply-chain-attack) — Payload analysis, IoC details, recommended actions
- [GitHub Security Advisory: Malware in 42 @tanstack/* packages exfiltrates cloud credentials, GitHub tokens, and SSH keys (GHSA-g7cv-rxg3-hmpx)](https://github.com/TanStack/router/security/advisories/GHSA-g7cv-rxg3-hmpx) — Official advisory with affected/patched versions
- [Socket Security: pnpm 11 Adds Supply Chain Protection Defaults](https://socket.dev/blog/pnpm-11-adds-new-supply-chain-protection-defaults) — Context on how package managers are hardening against these attacks