# LaunchDarkly-controlled Shared Actions demo

Demonstrates routing a CI job to a specific version of each shared GitHub composite action based on LaunchDarkly flag values. The team that owns the shared actions can roll out new versions progressively (1% → 5% → 100%) without consuming teams editing anything.

## Layout

- `.github/actions/sast-scan/` — first shared action (the versioned one)
- `.github/actions/secret-scan/` — second shared action (the versioned one)
- `.github/workflows/ci.yml` — consumer CI workflow

A separate flag controls each action's version, so the two roll out independently. The workflow uses [`jenseng/dynamic-uses`](https://github.com/jenseng/dynamic-uses) as a dispatcher because GitHub Actions resolves step-level `uses:` refs at workflow-parse time and rejects expressions referencing `steps`, `needs`, `inputs`, or `env` there (a security feature for org-level third-party-actions policies). The dispatcher's own `uses:` is static, but its `with:` block accepts any context, and it invokes the target action dynamically at runtime. For enterprise deployment, pin the dispatcher to a specific SHA, or fork/replicate it inside the central-IT repo.

## How it works

When CI runs:

1. The `launchdarkly/gha-flags` step evaluates `sast-scan-version` and `secret-scan-version`, passing a per-run context key (composite of `GITHUB_REPOSITORY_ID`, `GITHUB_RUN_NUMBER`, and `GITHUB_RUN_ATTEMPT`) so percentage and guarded rollouts randomize per CI job run — not per consumer team or repository. (FN1)
2. LD evaluates targeting rules using the auto-included GitHub metadata (repository, ref, actor, etc.) plus `LD_appid`, a custom attribute the consumer sets to identify the calling application/team.
3. Each subsequent step invokes the `jenseng/dynamic-uses` dispatcher with a `uses:` parameter built from the action name prefix and the LD-returned version: e.g., `EmilHorne/gh-shared-actions-demo/.github/actions/sast-scan@sast-scan-${{ needs.evaluate-flags.outputs.sast-scan-version }}` resolves to `sast-scan-v1`, `sast-scan-v2`, or `sast-scan-stable` depending on what LD returned. Each action's refs are versioned and advanced independently. (FN2)

(FN1) **Randomization is scoped, not global.** LD rollouts have two independent levers: *targeting rules* define **eligibility** (which CI runs can participate in the rollout at all), and the *randomization unit* assigns variations **within the eligible pool**. To restrict eligibility by repository name, target on the `Github.key` attribute (set by `gha-flags` from `GITHUB_REPOSITORY`); to restrict by consuming team or application, target on `GithubCustomAttributes.appid` (set from the `LD_appid` env var). Variations are then distributed across the eligible CI runs using `GithubCustomAttributes` as the randomization unit, keyed by the per-run `CI_JOB_ID`. A rollout might say *"of all CI runs from `appid in {team-a, team-b}`, send 10% to v2"* — eligibility limits the pool, randomization picks who within the pool gets the new variation.

(FN2) If both the central IT-operated LD Relay Proxy tier and LD SaaS are unreachable from the runner, the inline `,stable` default keeps each consumer on the version the central team has currently marked stable *for that action* — no surprise CI failures, and no edits to this file when any action's floor moves. See "Alternatives and FAQ" below for the full failure-mode architecture.

## One-time setup

1. **LaunchDarkly:** create two string flags — `sast-scan-version` and `secret-scan-version` — each with at least a `v1` variation, default rule `v1`. Values are version suffixes only (`v1`, `v2`, `stable`); the workflow concatenates the action-name prefix.
2. **Repo secret:** add `LD_SDK_KEY` (server-side SDK key for the LD environment).
3. **Per-action tags and stable branches.** With the initial files committed, create a `v1` tag and a `stable` branch *for each action*:
   ```
   git tag sast-scan-v1
   git branch sast-scan-stable
   git tag secret-scan-v1
   git branch secret-scan-stable
   git push origin sast-scan-v1 sast-scan-stable secret-scan-v1 secret-scan-stable
   ```
   Each `<action>-stable` branch is fast-forwarded independently by the central team as new versions of that specific action are validated. It is the inline fallback for that action when LD is unreachable.

## Demonstrating progressive rollout

1. Edit `.github/actions/sast-scan/action.yml` — change the echo to read `v2`. Commit and tag `sast-scan-v2`.
2. In LD, add a `v2` variation to `sast-scan-version` and set a percentage rollout (e.g. 10% v2 / 90% v1), optionally targeting by `appid`, `repository`, or any other attribute. The `secret-scan-version` flag is untouched — `secret-scan` remains at v1 for everyone.
3. Trigger CI runs. As LD targeting changes, runs pick up `sast-scan-v2` without anyone editing the workflow, while `secret-scan` continues at `secret-scan-v1`. The two actions roll out independently.
4. To roll back: flip the `sast-scan-version` flag back to 100% v1. Next run uses `sast-scan-v1`. `secret-scan` is unaffected throughout.

## Notes for real use

- In production, the shared actions would live in a central-IT repo (e.g. `enterprise-platform/shared-actions`), and consuming repos would reference `enterprise-platform/shared-actions/.github/actions/sast-scan@sast-scan-${{ steps.ld.outputs.sast-scan-version }}`. Same pattern, different `uses:` path. An alternative layout some enterprises prefer is one repo per shared action (`enterprise-platform/sast-scan`, `enterprise-platform/secret-scan`), with each repo using plain `v1`, `v2`, `stable` refs — that avoids the prefix concatenation entirely at the cost of more repos to operate. Pick based on whether the central team prefers a single substrate or per-action repository isolation.
- `LD_appid` is hardcoded here to `demo-app-team-a`. Real consumers would set it from a repo or org variable so LD can target by team.
- **Advancing per-action `stable` branches.** After a release of a specific action reaches 100% in LD and bakes long enough to be trusted, the central team fast-forwards just that action's floor:
  ```
  git checkout sast-scan-stable
  git merge --ff-only sast-scan-v2
  git push
  ```
  Fast-forward only — no rewritten history, no force push, and `sast-scan-stable` always points at a real tagged release commit. Each action's `<action>-stable` branch is advanced on its own schedule; `secret-scan-stable` is unaffected by the move above. Branch protection rules on each `<action>-stable` branch (require linear history, restrict who can push) keep this disciplined at scale.
- **`@<action>-stable` works identically for branches and tags.** GitHub's `uses: owner/repo/path@<ref>` syntax resolves any ref name — branch, tag, or full SHA — through the same lookup. Consumers don't need to know (and shouldn't care) whether the central team implemented each action's floor as a branch or a movable tag; the workflow file is unchanged either way. This is what makes switching implementations later a central-team concern only.

## Alternatives and FAQ

**Q: What's the runtime audit trail? For regulated builds we need to know which version of every shared action actually ran.**
The "Show selected versions" step prints the resolved values to the job log, retained per your GitHub log retention policy. For stronger guarantees, append the resolved versions (and the SHAs they resolved to) into the build's SBOM or attestation so the version-in-effect is preserved alongside the artifact, not just in ephemeral logs.

**Q: A regulated release is under a change-freeze. Can a consuming team freeze at the versions they are currently receiving?**

A consuming team in a freeze wants the versions *they* were resolving to the day the freeze started — not whatever the central team has designated as a freeze variation. Asking the central team to maintain opt-out variations in lockstep with partial rollouts puts responsibility for freeze correctness in the wrong place and is error-prone on both sides. The right framing is to make freeze-time evaluation a property of the consumer's workflow, not the central config.

Today, this could be solved by deploying LaunchDarkly Relay Proxies in offline mode with a config snapshot taken at the start of the freeze, ensuring the same snapshot is used throughout the freeze regardless of any changes to the central config. The problem for our use case is that each consuming team has to operate its own snapshot cache (a relay proxy instance) — exactly the per-team infrastructure pattern we want to avoid.

A cleaner path forward is an enhancement opportunity for `launchdarkly/gha-flags` itself. The package wraps the LaunchDarkly Node.js Server SDK, which already supports [reading flag values from a file](https://launchdarkly.com/docs/sdk/features/flags-from-files). Adding a `source-file` option to the GitHub Action wrapper is a small extension. The CI workflow would conditionally use file-mode only when a freeze is in effect — detected from an org-level variable, a calendar lookup, or a freeze API endpoint. Outside a freeze, the workflow operates normally and writes the resolved values to a committed file (`ld-flag-values-for-ci.json`) so a current snapshot is always available the moment a freeze starts. No per-team infrastructure; the snapshot lives in the consumer's own repo, owned by the consumer.

The file format expected by the SDK is:
```json
{
  "flagValues": {
    "sast-scan-version": "v1",
    "secret-scan-version": "v1"
  }
}
```

The demo workflow already includes the snapshot-refresh half — see `Refresh change-freeze snapshot` in `.github/workflows/ci.yml`. It runs only on `push` to `main` so PR runs don't pollute the snapshot, regenerates the file from the resolved outputs, and commits if changed. The freeze-mode *read* side of the enhancement is the collaboration ask for the `gha-flags` maintainers; the *write* side runs today with the script shown.

**Q: Who can change these flag values, and how does that fit existing change management?**

Flag changes are a privileged operation governed by the same type of controls as releases: approval workflows (integrated with ITSM policy and systems of record) gate changes to production flag configuration, and the LD audit log feeds into your SIEM via webhooks or direct integrations.

Treat the flag change — not the deployment of a new version — as the release, and govern it accordingly. The intention of using a flag is to replace a single big-bang new-version event with multiple new-version events delivering the same version to successive cohorts. The highest risk is in the later changes, not the earlier ones, as the size of the impacted audience scales up. It is therefore optimal to concentrate risk reviews at the *beginning* (was this tested?) and at the *end* (is this ready for general availability?) of the rollout, allowing the intermediate steps (e.g. 5%, 10%, 25%) to occur with minimal friction — ideally with fully automated approvals referencing the original approval of the initial-audience release, while still respecting timing factors like change freezes.

Two implementation paths exist for governing flag changes, and the choice has real downstream consequences:

*Terraform-managed flag configuration.* All flag state lives in version-controlled HashiCorp Configuration Language (HCL), reviewed via PR. Every change — including each step-wise percentage increase — is its own PR with a full approval audit trail. Strong fit for organizations that already centralize platform config in Terraform and want a single substrate for both infrastructure and release control. The cost is friction at every step: a 1% → 5% → 10% → 25% → 100% rollout becomes five PRs, each blocking on review. It also rules out LD-native release automation, particularly guarded rollouts, since those mutate flag state outside of Terraform's control plane.

*LD-native release automation, particularly guarded rollouts.* [Guarded rollouts](https://launchdarkly.com/docs/home/releases/guarded-rollouts) (Guardian plan) randomly assign a new flag variation to a percentage of contexts — in our case, CI jobs that are themselves already opted in via the phased rollout progression — and monitor chosen metrics for regressions. For this use case the relevant metrics are events like "CI job failed" or "CI step failed", published back to LD by each consuming team's CI workflow (see the demo workflow's `Publish failure event to LD` steps in `.github/workflows/ci.yml` for the format). LD correlates failure rates against flag variations using sequential statistical testing, and when it detects a statistically significant regression in the new variation it automatically rolls back to the prior variation for everyone in the current rollout phase and notifies the flag's maintainers. This collapses MTTD (mean time to detect a regression) from the hours-or-days of human triage to the minutes of automated comparison.

For correlation to work, the guarded rollout in LD must be configured with **`GithubCustomAttributes` as its randomization unit**. This is the context kind under which `launchdarkly/gha-flags` registers the per-run context key (`CI_JOB_ID`) — see [`src/action.js`](https://github.com/launchdarkly/gha-flags/blob/main/src/action.js) in that repo, which constructs a multi-context with three kinds: `Github` (repository-scoped), `GithubRunner` (runner-scoped, often empty), and `GithubCustomAttributes` (the kind that carries our context key and the `LD_appid` attribute). The failure events in the demo workflow set `contextKeys` to `{"GithubCustomAttributes": "<CI_JOB_ID>"}` so they join to the same kind/key pair that was used at evaluation time. If a different randomization unit were configured for the rollout, the failure events would still arrive in LD but wouldn't correlate to the variation served, and auto-rollback would not trigger.

The regression-detected event can also feed automated remediation. LD's [Vega](https://launchdarkly.com/docs/home/observability/vega) observability agent (currently in Early Access) has a GitHub integration that, when paired with the Vega + GitHub Copilot workflow, ingests surrounding logs, traces, and error data, identifies the likely root cause, and (in Fix mode) opens a pull request against the shared-action repo with a proposed fix. The central team reviews and merges; the loop continues without manual triage of every regression.

The net effect of the LD-native path is that the only human-in-the-loop approvals are the ones that genuinely warrant them: the initial release (was it tested?) and the GA promotion (is this ready for everyone?). The intermediate steps run automated and self-healing under the umbrella of the original approval, bounded by the same change-freeze calendar.

**Q: What's the failure mode if the LD SaaS endpoint is down?**

The architecture is layered, so most failures never reach the inline fallback. Central IT operates a pair (or more) of LD Relay Proxies behind a load balancer, each backed by a persisted database (Redis, DynamoDB, Consul, etc.). The relays sync with LD SaaS on a best-effort basis and persist flag config; the CI workflow's `gha-flags` step is configured to connect to the load balancer rather than LD SaaS directly, via the `base-uri`, `stream-uri`, and `events-uri` inputs (see the commented-out production configuration in `.github/workflows/ci.yml`).

This gives three independent layers of resilience: if LD SaaS is degraded, the relays continue serving from persisted state; if an individual relay node fails, the load balancer routes around it (and the restarted node reloads recent config from its backing database without needing SaaS); and only if *both* the entire relay tier and LD SaaS are unreachable from the CI runner does the workflow fall through to the inline `,stable` default — itself a valid ref to a real tagged release.

The LD step is in the critical path of every CI run across many consumer teams, which is why this layered fallback isn't optional. It's the whole reason the pattern is acceptable in regulated CI.

**Q: What's the cost and latency overhead?**

Latency is ~200ms or less per CI run, dominated by `gha-flags` initializing the SDK and fetching flag configuration. Once initialized, every flag evaluation happens in-memory inside the SDK — no further network calls per flag, regardless of how many flags the workflow checks. With the production deployment pattern (`gha-flags` pointed at the central IT relay tier rather than LD SaaS), the init fetch is an internal-network hop to a relay serving from its in-memory cache, not a call out over the internet to LD SaaS, so the latency budget is dominated by intra-datacenter networking. LD pricing is typically per-seat/MAU rather than per-evaluation, so CI volume specifically is rarely the cost driver — confirm with your contract.

**Q: What does the status quo actually cost?**

Worth measuring rather than assuming. A simple baseline: one shared-action update per month breaks CI for ≥10 consuming teams; each spends ~1 hour overriding and later backing out the change at $100/hr loaded ($12,000/year), and one team per month loses ~30 minutes of CI productivity at $100/minute ($36,000/year). That alone is **~$48,000/year** — but it's the tip of the iceberg.

Factors commonly missing from the baseline:

- **Triage time before the fix decision.** Across affected teams, ~2 hours of diagnosis per incident before the shared action is identified as the cause: **~$24,000/year**.
- **Central team coordination overhead.** Triage, communication, decision-making, executing rollbacks: **~$5,000–15,000/year**.
- **Lost improvements during a `stable` revert.** When the central team rolls the floor back to an older version to unblock the worst-affected teams, the entire consumer population temporarily loses the interim bug fixes, security patches, and capabilities. Carries forward as technical debt and (in regulated environments) extended exposure on known issues.
- **Bypassed-CI defect leakage.** Teams under deadline pressure bypass CI gates. If ~5% of bypassed runs would have caught a real defect at $5,000–50,000 production cost each, expected annual cost is in the tens of thousands to low six figures.
- **Production-incident MTTR amplification.** When CI is broken during a production incident, fix time extends. Production downtime cost is typically an order of magnitude higher than CI downtime — e.g., 2 incidents/year × 30 minutes added × $1,000/minute = **~$60,000/year**.
- **Delayed feature releases.** One release per month delayed by one day at $10,000/day of revenue/competitive value: **~$120,000/year**.
- **Central team opportunity cost.** 15–20% of a central tooling team's capacity consumed by incident coordination instead of new value: roughly **one FTE-loaded equivalent (~$150,000+)** per year.
- **Compliance findings.** Each audit finding tied to CI bypass or version-control gaps: **$10,000–50,000** to close, plus indirect audit-narrative cost.

Conservative sum (baseline + triage + central coordination): **~$80,000/year**. Plus moderately likely factors (bypass risk, prod amplification): **$150,000–200,000/year**. Including high-leverage factors (delayed releases, opportunity cost): **>$400,000/year**. LD's enterprise pricing typically sits below the low end of this range — and serves many use cases beyond CI versioning, so the marginal cost allocated to this use case is even lower.

To replace estimates with measurements for a specific organization:

- **Incident-management system** (ServiceNow / Jira / PagerDuty): count tickets tagged as CI breakage over the last 12 months and MTTR distribution per ticket. Often the single best signal.
- **Git history across consumer repos**: search commit messages and PR titles for `pin`, `revert`, `bypass`, `skip ci`, `workaround`, `temporarily disable`, `hardcode version`. A simple script gives a defensible count of workaround events.
- **CI platform metrics**: GitHub Actions / GitLab / Jenkins APIs expose workflow failure rates over time; correlate spikes with shared-action release dates to derive breakage rate directly.
- **One-time engineering survey** (5 minutes): ask consuming teams how often shared-action changes caused rework in the last year and estimated person-hours. Self-reports undercount by 30–50%; apply a multiplier.
- **Slack/Teams channel analysis**: message volume in `#ci-help`, `#shared-actions`, etc., filtered for `broken`, `rollback`, `stuck`, `help`. Rough counts are still informative.
- **Production-incident post-mortems**: scan for any incident where CI was a contributing or amplifying factor. These are usually the most expensive individual data points.
- **Audit findings register**: count findings tied to release controls, CI gate bypass, or version provenance over the last 1–3 audit cycles.

Convert to dollars using loaded rates appropriate to the organization: $100–150/hour for engineering time, $500–10,000/minute for production downtime depending on revenue scale, $10,000–50,000 per audit finding closure.

### Alternative strategies considered

| Approach | Pros | Cons |
| --- | --- | --- |
| **Floating major-version tags** (`@v1` auto-updates to latest v1.x, the `actions/checkout@v4` model) | No external dependency; well-understood; free | All-or-nothing rollout; no targeting or percentage exposure; rollback requires retagging (force-pushed tags break consumers mid-flight) |
| **Manual version bumps via PRs to each consumer repo** | Full git audit trail; reviewable by each team | Requires every consuming team to act — this *is* the problem; MTTR measured in days; emergency rollback needs N coordinated PRs |
| **Dependabot / Renovate auto-PRs** | Standard tooling; PR review trail per team | Still consumer-team-merge-gated; no rollback story; can't do percentage rollout or targeting |
| **Central versions-of-record config file synced into each consumer repo** | Auditable; no SaaS dependency | Still N PRs per change; pace bottlenecked on slowest consumer team |
| **GitHub Environments + deployment protection rules** | Native to GitHub; strong audit | Solves *who can deploy*, not *which version runs in CI* — different problem; doesn't address MTTR |
| **Internal private actions registry with curated promotions** | Strong governance; air-gap-friendly | Heavy infrastructure to build/operate; doesn't itself enable percentage rollout or per-team targeting (you'd end up reimplementing flag logic inside the registry) |
| **OpenFeature / other vendor** in place of LaunchDarkly | Same architectural pattern; some are self-hostable | No first-party GitHub Action equivalent to `gha-flags` for most — would need a custom composite action wrapping the SDK |

### When this approach fits — and when it doesn't

**Fits when:**
- Shared actions change frequently enough that MTTR is a real metric.
- Owning team and consuming teams are organizationally distinct (central IT vs. many app teams).
- You already operate LaunchDarkly (or comparable) and have the cost amortized across other use cases.
- Per-team, per-environment, or percentage targeting is a real requirement, not theoretical.
- You want regressions in new shared-action versions to be **auto-detected and self-healed** via guarded rollouts and (optionally) Vega-style auto-remediation, pushing MTTD and MTTR for the common failure modes below the threshold of human intervention.
- You want consuming teams to have a **fast, audited, self-service break-glass path** that doesn't queue behind the central team's response time. Implementable today: give consumers LD-portal access scoped to authoring [individual targeting rules](https://launchdarkly.com/docs/home/account/roles/role-actions#expand-feature-flag-actions) (`updateTargets`) on the relevant flags — i.e., the ability to add a target of the form "for `Github.key = my-repository`, serve `v1`" (targets match the `key` attribute of a context kind, which in our setup carries the repository name) — gated by [LD Custom Approvals](https://launchdarkly.com/docs/integrations/custom-approvals) for automated approval workflows, or audited after the fact via [webhooks](https://launchdarkly.com/docs/home/infrastructure/webhooks) on `updateTargets` events.
- You want the flag-evaluation path itself to be highly available: the relay-tier + LD SaaS + inline `,stable` default architecture means total flag-service unavailability requires multiple simultaneous failures.

**Doesn't fit when:**
- You need cryptographic immutability of CI versions per release. Reconciliation: combine flag-pick at runtime *and* record the resolved SHA in the build attestation — but if the audit regime forbids the runtime indirection itself, this isn't the right tool.
- You don't already operate a feature flag platform, your organization is small, and the status-quo cost (see the FAQ entry above) is below the platform-investment threshold for your size. For most mid-to-large enterprises this bar is lower than expected; worth running the numbers before defaulting to "we don't have LD, so this is out."

**Cases where alternatives look cheaper but trade their visible cost for a hidden one:**

*"Shared actions are stable; updates are rare."* Update frequency doesn't change blast radius — it only changes when you feel it. A single bad update breaks CI for every consumer at once, and the PR-based recovery path takes hours to days depending on consuming-team availability. The hidden cost surfaces when CI breaks during a production incident — exactly when shipping a fix is most urgent — because the operational fallback is usually "skip CI" or "bypass CI gates." That trades a pipeline outage for a much more serious compliance and audit problem, on the worst possible day. A flag revert is surgical, scoped, and leaves the audit trail intact; bypassing CI is fire-with-fire.

*"We only have a handful of consuming teams."* Few teams reduces frequency, not severity. Even one broken team during a critical incident motivates this design. The "PRs are cheaper than runtime flag evaluation" argument assumes recovery time isn't a constraint — until it is. With guarded rollouts, an unknown share of regressions are auto-reverted before any consuming team is aware of them; without it, every regression becomes a manual escalation regardless of team count. The central-team-bottleneck objection ("now we depend on central IT to flip the flag") inverts when consumers have audited self-service portal access: in the flag-versioning model, consuming teams have a fast self-service path during incidents; in the PR-versioning model, they don't — they have a slow path through their own repo's review process, or the much riskier path of bypassing CI entirely.

This pattern isn't universally "best." It's optimized for one specific failure mode: an update to a centrally-owned shared action breaks CI across hundreds or thousands of consumer teams, and recovery time depends on those teams each modifying their workflows. For an enterprise's central tooling team in exactly that situation, the alternatives either fail to address MTTR (manual PRs, Dependabot) or don't provide progressive exposure (floating major-version tags, internal registries without flag logic). The LD-driven approach trades a runtime dependency and ~200ms of CI overhead for the ability to roll forward and back in seconds, centrally, without touching consumer repos.
