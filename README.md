# Open Source Fix Analysis

This repository records upstream fixes and independently verifiable contributions incorporated into open-source projects. Each entry links the original review, the change, and the validation evidence. PRs integrated through internal code-sync are identified explicitly rather than represented as GitHub-native merges.

## Method

Inventory → Discover → Validate → Assign → Remediate → Prove

For executable bugs, I seek a reproducible failure on the upstream base, a narrow fix and a regression demonstrating the difference. Documentation and configuration corrections are checked against the source of truth. Impact ratings are self-assessed, not maintainer-assigned; an API panic is not automatically a remote denial-of-service vulnerability.

## Merged fixes

### [EthSystems Works #51 — rotortree snapshot root panic](https://github.com/ethsystems/works/pull/51) · Medium API reliability (self-assessed)

- **Problem:** A valid `rotortree` Merkle tree can grow to `MAX_DEPTH`, but `TreeSnapshot` stores only `MAX_DEPTH` non-root level slots in `levels`. Calling the public `level_len(depth)` or `get_node(depth, index)` at that depth indexed `levels[MAX_DEPTH]` and panicked, including before the tree reached full leaf capacity. This is a reproducible reliability defect; a production outage or remote exploit has not been demonstrated.
- **Fix:** Return the root count and hash from the separately stored `self.root` whenever `level == self.depth`, preserving empty-tree behaviour and returning `IndexOutOfRange` for invalid indices. The patch touches only the two accessors and their tests in `crates/rotortree`.
- **Proof:** The maximum-depth regression failed with an index-out-of-bounds panic against upstream `e5061c8` (1 failing and 3 passing focused tests); applying the patch gave **4 passing, 0 failing**. Regression coverage exercises sequential and batch insertion, binary depth 1 and 3, ternary depth 2, and each maximum-depth leaf-count range, plus empty/single-leaf controls. Locally, **107 default**, **180 all-feature** and **107 no-default-feature** tests passed, alongside Clippy, nightly formatting and diff checks. Maintainer `rymnc` requested test simplification and a signed commit, approved the revised patch, and merged it on 13 October 2026 as [`1449c62`](https://github.com/ethsystems/works/commit/1449c629b33e312d56b3a2f8e4511b69878524ba).

### [DeFiLlama SDK #266 — log-cache range union](https://github.com/DefiLlama/defillama-sdk/pull/266)

- **Problem:** A forced nested `getLogs` fetch could reduce the recorded end of a cached range. With an existing [100, 1000] range, requesting [500, 600] wrote [100, 610], because the merge replaced `toBlock` with the shorter padded range's end.
- **Fix:** Keep `Math.max` of covered end blocks when ranges overlap; merge adjacent ranges while leaving disjoint ranges separate.
- **Proof:** A red/green Jest regression exercises nested, right-extending, adjacent and disjoint ranges. **4 tests**, the TypeScript build and diff check passed. Maintainer `g1nt0ki` approved; merged 12 October 2026 as [`f286359`](https://github.com/DefiLlama/defillama-sdk/commit/f286359540579f4f1a9f41366182f8fab3519e2e). This fixes cache bookkeeping; no missing-log incident is claimed.

### [WhatsApp Erlang Language Platform #284 — grouped Common Test runnables](https://github.com/WhatsApp/erlang-language-platform/pull/284)

- **Problem:** Rebar3 Common Test runnables emitted a bare group name after `--case` instead of the required `--group <name>`.
- **Fix:** Insert the flag only for grouped tests, leaving ungrouped runnables unchanged; add both grouped and ungrouped regressions.
- **Proof:** The new regression failed before the fix while **1,809 control tests passed**. Workspace tests passed in reported Linux and macOS OTP matrices. **Meta internal merge:** `robertoaloi` imported the contribution into internal review and merged it as [`05020fd`](https://github.com/WhatsApp/erlang-language-platform/commit/05020fda13fa6592538fb56ca2b1d463ab3e0c8f), as [confirmed by Meta's code-sync bot](https://github.com/WhatsApp/erlang-language-platform/pull/284#issuecomment-6078526521). GitHub marks the original PR closed rather than natively merged.

### [Nethermind Juno #4146 — JSON-RPC notification responses](https://github.com/NethermindEth/juno/pull/4146)

- **Problem:** Juno sent JSON-RPC error responses to requests without an `id` when the method was unknown or parameters invalid. The JSON-RPC specification prohibits responding to notifications.
- **Fix:** Suppress all notification responses after dispatch without suppressing error responses for ID-bearing requests.
- **Proof:** Four red/green regressions cover unknown-method notifications, invalid-parameter notifications, notification-only batches and mixed batches. The relevant packages passed `go test -race`. Approved and merged 8 October 2026 as [`4c6b4d3`](https://github.com/NethermindEth/juno/commit/4c6b4d318a6eb038fabb918db7cf271763827c80).

### [Polygon Agent CLI #142 — x402 dry-run transaction safety](https://github.com/0xPolygon/polygon-agent-cli/pull/142)

- **Problem:** `x402-pay` unconditionally broadcast a funding transaction, bypassing the CLI's persisted dry-run transaction mode.
- **Fix:** Use the existing transaction-mode resolver and shared `--broadcast` / `--dry-run` flags; return a structured payment preview rather than moving funds while dry-run is active.
- **Proof:** Regression coverage checks standard x402 and Bazaar payment paths do not submit transactions in dry-run mode. [Fixes issue #130](https://github.com/0xPolygon/polygon-agent-cli/issues/130); merged 8 October 2026 as [`b3e81d0`](https://github.com/0xPolygon/polygon-agent-cli/commit/b3e81d0fa64b95a5c83f0a1073ee9ea30ec2b6d8).

### [Solana Foundation website #2150 — transaction v1 slot-time blocks](https://github.com/solana-foundation/solana-com/pull/2150)

- **Problem:** The slot-time endpoint requested full blocks with `maxSupportedTransactionVersion: 0`; blocks containing v1 transactions could be rejected, leaving a stale cached block instead of a fresh update.
- **Fix:** Permit version 1 in the RPC request and preserve program-ID resolution using static and loaded address-table keys.
- **Proof:** Regressions assert the exact JSON-RPC request and address resolution; **3 Vitest tests**, TypeScript and formatting checks passed. [Fixes #2141](https://github.com/solana-foundation/solana-com/issues/2141); merged as [`a233795`](https://github.com/solana-foundation/solana-com/commit/a233795827fb9ff3581a3f915c7ccc518c1a7b88).

### [Promptfoo MCP Agent Provider #95 — measured model usage](https://github.com/promptfoo/mcp-agent-provider/pull/95)

- **Problem:** The provider estimated OpenAI token counts from message counts and fabricated costs with a hard-coded price, producing misleading usage reports across multi-step tool calls.
- **Fix:** Aggregate actual API-reported token usage across ReAct iterations, omit incomplete totals, and remove unsupported synthetic costs.
- **Proof:** Real-SDK loopback regressions cover measured, zero and missing usage; the reported **31 tests** and Node 20/22/24 checks passed. Merged upstream as [`6478386`](https://github.com/promptfoo/mcp-agent-provider/commit/6478386ffb7207ab832d0e0849695213e09bef71).

### [Promptfoo MCP Agent Provider #96 — legacy SSE fallback](https://github.com/promptfoo/mcp-agent-provider/pull/96)

- **Problem:** The advertised Streamable HTTP → SSE fallback ran only during transport construction, but incompatibility with an SSE-only endpoint surfaces during `client.connect()`.
- **Fix:** On qualifying 4xx `StreamableHTTPError` negotiation failures, close the failed connection, create a fresh MCP client and retry with legacy SSE; do not mask unrelated transport failures.
- **Proof:** An end-to-end legacy SSE server fixture validates the negotiated fallback, with Node 20/22/24 and Biome CI reported passing. Merged as [`bf5f668`](https://github.com/promptfoo/mcp-agent-provider/commit/bf5f6686bb62c16c73b13bf0220c7205263b66bf).

### [OpenAI Codex Security #1306 — Vyper source scan coverage](https://github.com/openai/codex-security/pull/1306)

- **Problem:** The shared source-code allowlist omitted `.vy` files, so Vyper-only changes could disappear before scan inventory and ranking.
- **Fix:** Include Vyper sources in existing text-code inventory and ranking flows and add targeted regressions for repository, revision-diff and local-patch paths.
- **Proof:** An OpenAI maintainer independently reproduced **five failing inventory/ranking cases on the parent** and verified that all passed after the fix. **295 tests** passed across affected modules, with no blocking behaviour issues. Merged 5 October 2026 as [`d327164`](https://github.com/openai/codex-security/commit/d32716411f3d7b87085b36f9de55f0ac40454c42).

### [Nethermind #14228 — JWT cache expiration](https://github.com/NethermindEth/nethermind/pull/14228)

- **Problem:** A correctly signed JWT that initially passed validation could continue to authenticate from the cache after its explicit `exp`, because cache hits used the `iat` window without rechecking expiration.
- **Fix:** Apply the same exclusive expiry boundary (`now >= exp` is rejected) to cached and fresh validation while retaining the normal valid-token fast path.
- **Proof:** A deterministic clock-based reproduction compares fresh validation against cache reuse at `iat + 1`, exact expiry, and subsequent times. Both manual HS256 and library fallback paths are covered by regressions. Merged 5 October 2026 as [`eeca264`](https://github.com/NethermindEth/nethermind/commit/eeca26403bef8420a980ebde700c9e338caa75e8).


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
