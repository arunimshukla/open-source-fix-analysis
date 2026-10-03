# Open Source Fix Analysis

This repository records my upstream fixes that have been merged after reproducible validation. Each entry states the problem, the change and the proof kept with the pull request.

## Method

Inventory → Discover → Validate → Assign → Remediate → Prove

I include a finding only when it reproduces on the current main branch, has a narrow fix and has regression evidence.

## Merged fixes

### [Centrifuge API v3 #489](https://github.com/centrifuge/api-v3/pull/489)

- **Problem:** Pharos (chain ID `1672`) was listed in `networkNames` but missing from `explorerUrls`. The indexer therefore initialised `Blockchain.explorer` without a Pharos URL, causing Token and NAV links to fall back to the pool dashboard.
- **Fix:** Added `https://www.pharosscan.xyz` for chain `1672` in `src/chains.ts`, preserving the existing chain and RPC configuration.
- **Proof:** Checked the chain mapping and the timekeeper path that populates `Blockchain.explorer`. The merged diff adds only the explorer mapping. A maintainer approved the PR and merged it on 1 October 2026, closing [#479](https://github.com/centrifuge/api-v3/issues/479). Existing Pharos rows may still require a refresh or reindex because `getOrInit` does not update initialised records.

### [ethereum.org #19272](https://github.com/ethereum/ethereum-org-website/pull/19272)

- **Problem:** The Glamsterdam roadmap's manually maintained proposal notice had drifted from the generated upgrade data. It still listed EIP-7610 and omitted the scheduled networking proposals EIP-8070, EIP-8136 and EIP-8189.
- **Fix:** Removed EIP-7610 and added eth/72 (EIP-8070), EIP-8136 and snap/2 (EIP-8189) to the English source. The change preserves the page's structure and leaves translation propagation to the internationalisation pipeline.
- **Proof:** Checked the proposal list against `src/data/upgrades/generated.ts`. The PR records zero markdownlint errors and a passing `git diff --check`; its diff changes only `public/content/roadmap/glamsterdam/index.md`. A maintainer approved and merged the PR on 1 October 2026. The team also added me as an ethereum.org maintenance contributor in [#19364](https://github.com/ethereum/ethereum-org-website/pull/19364).

### [Google Chrome Platform Status #6946](https://github.com/GoogleChrome/chromium-dashboard/pull/6946)

- **Problem:** Chrome Platform Status hides stage-level actions when an API Owners gate is present and expects the gate column to render actions associated with that gate. The experiment-extension email action had an empty gate-type list in the normal launch, fast-track and deprecation process definitions, so neither rendering path displayed it.
- **Fix:** Associated all three email actions with `GATE_API_EXTEND_ORIGIN_TRIAL` and added one regression covering the three process definitions. The change leaves the email URL, prerequisites and approval rules untouched.
- **Proof:** The regression failed on the unpatched `main` branch for all three definitions with `AssertionError: 3 not found in []`, then passed after the fix. Ruff lint and formatting, the licence check, `git diff --check` and mypy across 363 source files also passed. A maintainer approved the change, and Google's merge queue landed it as [`ca80064`](https://github.com/GoogleChrome/chromium-dashboard/commit/ca80064d966c5f58df7c25941ebe8416d7ae9a03) on 29 September 2026. It was subsequently included in the closed [deployment release #6950](https://github.com/GoogleChrome/chromium-dashboard/issues/6950).

### [OpenAI Codex Security #1020](https://github.com/openai/codex-security/pull/1020)

- **Problem:** The `scan --patch` workflow could report success when the model returned `no_change` without explaining how it had checked the finding. A claim that no fix was needed could therefore clear the finding without verification evidence.
- **Fix:** Extended the existing verification requirement to `no_change` results. Missing or whitespace-only verification now produces a failed result and exit code 2. Valid `no_change` results remain accepted, and the response instructions explicitly require verification for both successful statuses. This checks that an explanation exists; it does not establish that the explanation is true.
- **Proof:** Controlled before/after tests reproduced success for missing and blank verification on the unpatched code and rejection after the fix. The [maintainer confirmed](https://github.com/openai/codex-security/pull/1020#issuecomment-5855914520) 72 focused tests passed, plus two full SDK runs with 3,242 passed, 50 skipped and no failures each. Builds, type checks, formatting and GitHub CI passed. Approved and merged on 27 September 2026, with the original fix credited to me.

### [OpenAI Guardrails JS #148](https://github.com/openai/openai-guardrails-js/pull/148)

- **Problem:** The vector-store helper checked a path's extension before checking whether the path was a file or directory. Directories such as `documents.v1` or `archive.zip` were therefore rejected even when they contained supported documents.
- **Fix:** Classified the path first, then applied the extension allowlist only to regular files. Directory filtering remains case-insensitive and non-recursive.
- **Proof:** Three dotted-directory regressions failed against the original implementation and passed after the fix. All 882 tests, build, lint and documentation checks passed locally; CI passed on Node.js 22, 24 and 26, together with both CodeQL analyses. OpenAI approved and merged the PR on 24 September 2026.

### [Promptfoo #11243](https://github.com/promptfoo/promptfoo/pull/11243)

- **Problem:** The Google Vertex Claude provider resolved `top_p` and `top_k` aliases with logical OR. Because `0` is falsy, an explicit zero could be replaced by the camelCase alias or omitted, changing the sampling configuration sent to the model.
- **Fix:** Replaced the two falsy fallbacks with nullish coalescing so only a missing value falls through, while preserving the existing alias precedence and newer-model sampling suppression.
- **Proof:** Regression cases cover explicit zero values against non-zero aliases. The maintainer independently verified that reverting either fallback makes its regression fail, then ran 1,146 Google provider tests, typecheck, the CLI build, formatting/lint and ten mocked-transport QA configurations. All 57 applicable PR checks were green before merge on 29 September 2026.

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
