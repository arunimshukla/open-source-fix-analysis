# Open Source Fix Analysis

This repository records my upstream fixes that have been merged after reproducible validation. Each entry states the problem, the change and the proof kept with the pull request.

## Method

Inventory → Discover → Validate → Assign → Remediate → Prove

I include a finding only when it reproduces on the current main branch, has a narrow fix and has regression evidence.

## Merged fixes

### [OpenAI Codex Security #1020](https://github.com/openai/codex-security/pull/1020)

**Requiring verification before accepting “no change needed”**

Merged on 27 September 2026. The [pull request](https://github.com/openai/codex-security/pull/1020) credits me with the original fix. The maintainer updated the branch, confirmed the validation results and approved the change before merge.

#### The problem in plain English

Codex Security's `scan --patch` workflow can ask a model to address a security finding. A successful outcome can mean that the model applied and checked a fix, or that it determined the existing code was already safe and needed no change.

Both conclusions need an explanation of what was checked. Before this fix, the application enforced that requirement for `verified` results but not for `no_change` results. A model could return “no change needed” without any verification explanation, and the workflow could still report success.

That matters when another system uses the command's result to decide whether a security check passed. The model's unsupported conclusion could be accepted as a resolved outcome. The regression demonstrated this with a high-severity finding and the command's severity-based failure policy; it did not demonstrate exploitation of a live system.

#### How the failure happened

The workflow reads a structured model response containing the finding identifier, result status, affected files and an optional verification explanation.

The relevant check originally applied only when the status was `verified`. A response with status `no_change` bypassed it. An omitted explanation and an explanation containing only spaces, tabs or line breaks both left the same gap.

The problem was therefore in the application logic that accepted the response. Updating the model's instructions alone would not enforce the requirement.

#### What changed

The fix extended the existing check to both successful statuses:

```ts
(parsed.data.status === "verified" ||
  parsed.data.status === "no_change") &&
!parsed.data.verification?.trim()
```

If this condition is true, the application converts the response into a failed result with the reason `Patch verification was not reported.`. The workflow returns exit code 2 instead of success.

The response instructions were also updated to require verification for both statuses. A valid `no_change` result remains supported: the model does not have to edit a file or create a pull request when it can explain why the current code is already safe.

The production change was limited to the existing validation check and response instructions. It introduced no new command options or output fields.

#### How the fix was proved

The regression tests supplied controlled model responses, allowing the same missing-verification cases to be exercised reliably without waiting for a live model to produce them.

| Response | Expected outcome after the fix |
| --- | --- |
| `no_change` with missing verification | Failed result; exit code 2 |
| `no_change` with whitespace-only verification | Failed result; exit code 2 |
| Valid `no_change` response | Remains accepted without creating a pull request |
| `verified` with missing or whitespace-only verification | Remains rejected |
| Blocked, failed or malformed response | Remains unresolved |

The [PR validation record](https://github.com/openai/codex-security/pull/1020) reports that the same regression tests failed twice on the unpatched main branch because missing and blank `no_change` verification returned success. All eight selected cases passed with the fix. The focused patch suite passed 72 tests using seed `12345` and the existing 30-second timeout.

The [maintainer's validation comment](https://github.com/openai/codex-security/pull/1020#issuecomment-5855914520) additionally confirms two full SDK runs, each with 3,242 tests passed, 50 skipped and no failures, plus successful builds, type checks, formatting and GitHub CI.

The maintainer also ran the built command with a real model against a benign local fixture. That scan completed with full coverage, no findings and unchanged source files. It checked scan compatibility; the controlled regression tests established the missing-verification failure and its correction. These results are attributed to the PR record and maintainer report, not to a new test run performed while writing this analysis.

#### What this fix guarantees

The application now refuses to accept either successful status without a nonblank verification explanation. It closes a specific gap in how model responses are accepted.

It does **not** prove that the explanation is correct, that the reported checks actually ran or that the underlying code is safe. A nonblank but inaccurate explanation can still satisfy this check. The contribution enforces a necessary evidence requirement; it does not independently validate that evidence.

The broader engineering lesson is that every outcome capable of clearing a finding needs an explicit acceptance rule. “No change needed” deserves verification just as an applied patch does.

### [OpenAI Guardrails JS #148](https://github.com/openai/openai-guardrails-js/pull/148)

- **Problem:** The vector-store helper checked a path's extension before checking whether the path was a file or directory. Directories such as `documents.v1` or `archive.zip` were therefore rejected even when they contained supported documents.
- **Fix:** Classified the path first, then applied the extension allowlist only to regular files. Directory filtering remains case-insensitive and non-recursive.
- **Proof:** Three dotted-directory regressions failed against the original implementation and passed after the fix. All 882 tests, build, lint and documentation checks passed locally; CI passed on Node.js 22, 24 and 26, together with both CodeQL analyses. OpenAI approved and merged the PR on 24 September 2026.

### [go-ethereum #35710](https://github.com/ethereum/go-ethereum/pull/35710)

- **Problem:** Ethereum JSON-RPC documentation no longer matched the implementation.
- **Fix:** Corrected block selectors, exclusions and response examples.
- **Proof:** Checked the documentation against the current RPC source.

### [Google Skill Reach #23](https://github.com/google/skill-reach/pull/23)

- **Problem:** CSV and JSONL round trips dropped query metadata and could change evaluation results.
- **Fix:** Preserved acceptable skills, notes and custom columns across both formats.
- **Proof:** Added round-trip coverage and passed the full test, type, lint and documentation checks.

### [NASA Delta #158](https://github.com/nasa/delta/pull/158)

- **Problem:** Cache eviction tried to remove regular files as directories, and the text-file exclusion used the wrong suffix.
- **Fix:** Removed files and directories with the correct operation and repaired the `.txt` check.
- **Proof:** Regression tests cover files, directories and protected CSV/TXT files.

### [Samsung CredSweeper #951](https://github.com/Samsung/CredSweeper/pull/951)

- **Problem:** CRX3 browser extensions were read with the older CRX2 header layout, so scans could start at the wrong ZIP offset.
- **Fix:** Added version-specific CRX2 and CRX3 parsing and rejected unsupported or truncated headers.
- **Proof:** Focused and scanner regression suites passed with lint and type checks.

### [EthSystems Map #200](https://github.com/ethsystems/map/pull/200)

- **Problem:** The ERC-3643 guide treated investor transfers, minting and administrative transfers as if they followed the same checks.
- **Fix:** Separated the transfer paths and clarified owner, agent, compliance and censorship assumptions.
- **Proof:** Compared the text with the canonical specification and reference implementation.

### [Consensys Ask O11y #226](https://github.com/Consensys/ask-o11y-plugin/pull/226)

- **Problem:** Newer OpenAI models rejected the legacy `max_tokens` request field.
- **Fix:** Replaced it with `max_completion_tokens` throughout request construction and diagnostics.
- **Proof:** A regression test requires the new field and rejects the old one.

### [Nethermind #13747](https://github.com/NethermindEth/nethermind/pull/13747)

- **Problem:** Engine API V3 and later accepted explicit null values for required payload fields and returned the wrong class of error later.
- **Fix:** Rejected null or omitted `withdrawals`, `blobGasUsed` and `excessBlobGas` as invalid parameters.
- **Proof:** Regressions cover explicit null and omitted fields. The maintainer PR superseded my #13734 while retaining my commits and authorship.

### [Nethermind #13755](https://github.com/NethermindEth/nethermind/pull/13755)

- **Problem:** Nethermind still served `engine_getPayloadV5` at Amsterdam, even though its V5 response cannot carry Amsterdam's `slotNumber` or `blockAccessList`. It could return an outdated payload shape instead of `-38005 Unsupported fork`.
- **Fix:** Limited V5 to its Osaka window and rejected it once block-level access lists are enabled.
- **Proof:** The focused regression failed in both fixture modes before the fix and passed 2/2 afterward. The V4/V5/V6 fork-boundary suite passed 33/33, preserving Osaka V5 behaviour. The PR received three maintainer approvals, merged into `master` on 24 September 2026 and closed [#13713](https://github.com/NethermindEth/nethermind/issues/13713) as completed.

### [DefiLlama Pegged Assets #927](https://github.com/DefiLlama/peggedassets-server/pull/927)

- **Problem:** The EURR description named Revolut as the issuer, although Bridge Building S.A. issues the token and Revolut distributes it.
- **Fix:** Corrected the issuer and distributor attribution.
- **Proof:** Cross-checked the wording against the issuer and distributor sources. No supply or pricing logic changed.

### [Dedaub srcup action #1](https://github.com/Dedaub/srcup-ci-action/pull/1)

- **Problem:** The workflow example used stale action versions and unsafe output handling.
- **Fix:** Updated the versions, explained the input/output key difference and used safe report printing.
- **Proof:** Checked the example against the published action interface.

### [Dedaub source warnings action #1](https://github.com/Dedaub/srcwarnings-ci-action/pull/1)

- **Problem:** The documentation used underscore input names that the action does not accept.
- **Fix:** Documented the correct hyphenated inputs, required API key and supported options.
- **Proof:** Checked both examples against the action definition.

### [Solana web3.js #3943](https://github.com/solana-foundation/solana-web3.js/pull/3943)

- **Problem:** Smoke tests did not execute IIFE browser bundles, so an unreplaced `process.env["NODE_ENV"]` reference could ship unnoticed.
- **Fix:** Corrected the replacement and made smoke tests execute both IIFE bundles.
- **Proof:** Production builds and CJS, ESM, browser, native and IIFE smoke tests passed.

### [Trail of Bits Skills #306](https://github.com/trailofbits/skills/pull/306)

- **Problem:** An obsolete configuration invoked the removed `codex mcp-server` command and could break Claude Code start-up.
- **Fix:** I reported and proposed the removal in #303. The maintainer shipped the core fix in the broader #306 clean-up.
- **Proof:** Plugin loadability, metadata, shell, Bats, JavaScript and temporary Git fixture checks covered the change.
