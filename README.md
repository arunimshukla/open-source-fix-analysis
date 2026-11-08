# Open Source Fix Analysis

This repository records upstream fixes and independently verifiable contributions incorporated into open-source projects. Each entry links the original review, the change, and the validation evidence. PRs integrated through internal code-sync are identified explicitly rather than represented as GitHub-native merges.

## Method

Inventory → Discover → Validate → Assign → Remediate → Prove

For executable bugs, I seek a reproducible failure on the upstream base, a narrow fix and a regression demonstrating the difference. Documentation and configuration corrections are checked against the source of truth. Impact ratings are self-assessed, not maintainer-assigned; an API panic is not automatically a remote denial-of-service vulnerability.

## Merged fixes

<a name="nethermind-juno-4199--websocket-busy-responses-for-notifications"></a>

### 01. [Nethermind Juno #4199 — WebSocket busy responses for notifications](https://github.com/NethermindEth/juno/pull/4199)

- **Problem** — Juno's WebSocket admission gate could return `-32004` with `id: null` for a valid JSON-RPC notification when the gate was full. The rejection happens before normal dispatch, so the notification handling in [#4146](https://github.com/NethermindEth/juno/pull/4146) never sees it.
- **Fix** — Suppress the busy write for a valid notification or an allowed, non-empty notification-only batch. Validate the complete JSON value and honour the batch enablement and size limits. ID-bearing requests, mixed batches and malformed input still take the busy-response path.
- **Proof** — A real WebSocket regression fills the gate and follows silent cases with an identified probe. The final batch-policy revision reports **37 passing cases**, with three new boundary cases failing on the preceding revision. The JSON-RPC race suite, `go vet` and lint checks passed. Merged on 3 November 2026 as [`4a498bf`](https://github.com/NethermindEth/juno/commit/4a498bf5d2e710c160d344dcb7519aa354a6ab7c).

<a name="fhenix-cofhe-sdk-455--stale-react-wallet-connections"></a>

### 02. [Fhenix CoFHE SDK #455 — stale React wallet connections](https://github.com/FhenixProtocol/cofhesdk/pull/455)

- **Problem** — `useCofheAutoConnect` skipped new client pairs while a connection was in flight. If the account or chain changed before the original attempt completed, CoFHE could stay bound to the old clients.
- **Fix** — Forward the latest public and wallet clients to the core SDK. Its existing attempt-ID guard prevents an older completion or error from replacing the newer connection.
- **Proof** — Five regressions use the real React provider and SDK store, with controlled wallet and RPC responses. Restoring the old guard makes three fail. All five pass with the fix, alongside **83 React tests**, **23 core client tests**, type checks and the React build. Merged on 29 October 2026 as [`4119d87`](https://github.com/FhenixProtocol/cofhesdk/commit/4119d875ff0a1788a541b8414909103614496186).

<a name="solana-idl-17--rpc-failures-during-history-reconstruction"></a>

### 03. [Solana IDL #17 — RPC failures during history reconstruction](https://github.com/solana-foundation/idl/pull/17)

- **Problem** — Signature-list failures could be converted to empty history, and transaction-fetch failures could be skipped during Anchor and Program Metadata Program replay. Callers could receive a successful but incomplete reconstruction.
- **Fix** — Remove the four error-suppression sites so the original RPC failure reaches the caller. Empty-history fallback, null transactions and failed-signature skipping still work.
- **Proof** — Five offline cases failed on the upstream base while five controls passed. All ten passed after the fix, along with **132 IDL tests** and **22 security.txt tests**. Recorded fixtures cover the fallback authority path. This follows the error-versus-absence handling in [#10](https://github.com/solana-foundation/idl/pull/10). Merged on 29 October 2026 as [`9bce612`](https://github.com/solana-foundation/idl/commit/9bce6120e00d9f44d3bb92b5eeaabd1c118c82d3).

<a name="nethermind-15190--pruned-receipts-for-non-canonical-blocks"></a>

### 04. [Nethermind #15190 — pruned receipts for non-canonical blocks](https://github.com/NethermindEth/nethermind/pull/15190)

- **Problem** — `eth_getLogs(blockHash)` could misclassify a previously processed non-canonical block with pruned receipts as unprocessed. Canonical retention by address and height also could not establish that receipts for the requested non-canonical hash were available.
- **Fix** — Consult the stored `WasProcessed` flag, check for missing non-canonical receipts before applying sliced-retention guarantees, and bypass the canonical height index for hash-pinned queries.
- **Proof** — Production-pruner regressions leave the orphan header and processing metadata in place while removing its body and receipts. Both log finders report unavailable history, and a separate indexed-query regression returns the orphan's logs when its receipts are available. The PR records **293 passing tests** across log finding, RPC and history pruning. Merged on 29 October 2026 as [`600db9b`](https://github.com/NethermindEth/nethermind/commit/600db9bf8f9516137d2cfff200f28de90101148e).

<a name="theorem-labs-oss-verify-9--allocation-instrumentation-and-conflicting-lock-pins"></a>

### 05. [Theorem Labs OSS Verify #9 — allocation instrumentation and conflicting lock pins](https://github.com/theorem-labs/oss-verify/pull/9)

- **Problem** — Allocation-failure and counting links could silently leave allocator calls untouched when libraries were supplied indirectly. Repeated `--lock` arguments also let the last hash override a conflicting pin for the same target and build.
- **Fix** — Require explicit object or archive inputs for the two allocation engines and reject conflicting SHA-256 pins before building. Ordinary `$LIB_FUZZING_ENGINE` links and identical or disjoint pins retain their behaviour.
- **Proof** — The maintainer reproduced **16 failures** on `main` and verified the regressions passed with the fix. Native linker fixtures exercise actual allocation failure and counting. The maintainer's suite reported **134 passing tests**, with Docker-dependent cases skipped. Landed through [#13](https://github.com/theorem-labs/oss-verify/pull/13) on 24 October 2026 as [`906db00`](https://github.com/theorem-labs/oss-verify/commit/906db00b97941e8823a5faf954de32c2cb7520c0).

<a name="theorem-labs-oss-verify-7--locked-commits-and-annotated-tags"></a>

### 06. [Theorem Labs OSS Verify #7 — locked commits and annotated tags](https://github.com/theorem-labs/oss-verify/pull/7)

- **Problem** — `tools/check --lock` resolved the live branch or tag before using the pinned commit, so a deleted or ambiguous ref could block a reproducible build. Fully qualified annotated tags could also resolve to the tag object instead of the commit the builder checked out.
- **Fix** — Use the pinned commit directly when present and peel qualified annotated tags to commits. Locks without a commit still resolve the live ref, and unqualified branch/tag ambiguity is still rejected.
- **Proof** — Real local Git fixtures cover deleted, moved and ambiguous refs plus lightweight, annotated and nested tags. The maintainer reproduced **5 failures** on `main` and verified **12/12** new cases passed after the fix. Docker-dependent cases were skipped in the broader run. Landed through [#12](https://github.com/theorem-labs/oss-verify/pull/12) on 24 October 2026 as [`e844fdd`](https://github.com/theorem-labs/oss-verify/commit/e844fdde05486a6fbbc5d9a5c57425b0e7a9d00f).

<a name="nethermind-14877--sendpolicy-rule-reloads-with-unchanged-metadata"></a>

### 07. [Nethermind #14877 — SendPolicy rule reloads with unchanged metadata](https://github.com/NethermindEth/nethermind/pull/14877)

- **Problem** — SendPolicy cached rules by file size and modification time. A replacement that preserved both could leave a removed destination permission active or hide malformed rules.
- **Fix** — Compare file contents and reread under the reload lock before publishing a change. Reuse the cached rules when the contents match. If the file becomes unusable, retain the last known guarded senders and refuse their sends. The configuration guidance now specifies atomic replacement.
- **Proof** — Eight same-metadata cases failed on the upstream base. A controlled concurrency regression also reproduced an older snapshot overwriting newer guarded-sender state on an earlier revision. The reported Linux run passed **77 tests** with one Windows-only test skipped. Merged on 21 October 2026 as [`dc6f0bc`](https://github.com/NethermindEth/nethermind/commit/dc6f0bc4266a1802286b79053377d31280797166).

<a name="solana-governance-242--top-voters-csv-export"></a>

### 08. [Solana Governance #242 — Top Voters CSV export](https://github.com/solana-foundation/solana-governance/pull/242)

- **Problem** — The Top Voters download control had no export handler. During review, the first implementation also left the control enabled during a background refetch and allowed formula-like validator names through.
- **Fix** — Export the complete filtered and sorted result before pagination. Escape CSV fields, neutralise formula prefixes in external strings and disable the download while votes are fetching or the result is empty.
- **Proof** — Regressions cover background refetches, missing stake accounts, CSV escaping and formula prefixes, including full-width operators. The PR records **434 passing Jest tests**, with **17** in the focused suites, plus TypeScript and ESLint checks. [Fixes #203](https://github.com/solana-foundation/solana-governance/issues/203). Merged on 20 October 2026 as [`bbcd837`](https://github.com/solana-foundation/solana-governance/commit/bbcd83758965903264615cde129bd67aee79a026).

<a name="nethermind-14372--malformed-receipt-post-state-lengths"></a>

### 09. [Nethermind #14372 — malformed receipt post-state lengths](https://github.com/NethermindEth/nethermind/pull/14372)

- **Problem** — Malformed receipt post-state fields could reach `Hash256` with an invalid length and throw `ArgumentException`. In peer-message decoding, that bypassed the normal RLP error handling.
- **Fix** — Check the length before constructing `Hash256` and throw `RlpException` for malformed post-state fields. The merged patch covers legacy and eth/69 message decoders, receipt storage and Optimism variants. Valid empty and 32-byte post-state fields retain their behaviour.
- **Proof** — Regression cases assert `RlpException` for malformed lengths, including 5, 31, 33 and 34 bytes in the message decoders. The merged diff includes Ethereum and Optimism coverage. The change puts already rejected inputs on the expected error path. Merged on 19 October 2026 as [`c1e371b`](https://github.com/NethermindEth/nethermind/commit/c1e371b633bac35e231cded69e4a9d9d3c6cae2b).

<a name="nethermind-14338--docker-udp-discovery-metadata"></a>

### 10. [Nethermind #14338 — Docker UDP discovery metadata](https://github.com/NethermindEth/nethermind/pull/14338)

- **Problem** — The standard and chiseled images declared `30303/tcp` but omitted `30303/udp`, which Nethermind uses for peer discovery. Automatic publishing with `docker run -P` therefore did not pick up the UDP port.
- **Fix** — Declare both protocols in both Dockerfiles and extend the existing image metadata check to require them.
- **Proof** — The PR records that both published images lacked the UDP declaration before the fix. Patched builds passed the exposed-port checks for TCP and UDP. This changes image metadata, while the client retains its existing runtime networking configuration. Merged on 16 October 2026 as [`0137e99`](https://github.com/NethermindEth/nethermind/commit/0137e997653badfd7365630f4bfbe48f58246b80).

<a name="ethsystems-web-50--aztec-privacy-model-in-the-faq"></a>

### 11. [EthSystems Web #50 — Aztec privacy model in the FAQ](https://github.com/ethsystems/web/pull/50)

- **Problem** — The FAQ grouped Aztec under account-based privacy. Aztec uses private notes, commitments and nullifiers alongside separate public contract state.
- **Fix** — Describe Aztec's hybrid model and limit the encrypted balance comparison to some FHE-based confidential-token designs.
- **Proof** — The original PR checked the wording against Aztec's state documentation and reports **63 passing tests**, including a focused case that failed on the original wording. The [maintainer confirmed the FAQ edit landed on `main`](https://github.com/ethsystems/web/pull/50#issuecomment-6165701204). The mirror carried the wording change and omitted the proposed test. Mirrored to `main` on 14 October 2026 as [`638a7a1`](https://github.com/ethsystems/web/commit/638a7a1dfd84fe65d592ba07cf5700f893694d4f).

<a name="ethsystems-map-203--trading-domain-use-case-links"></a>

### 12. [EthSystems Map #203 — Trading domain use-case links](https://github.com/ethsystems/map/pull/203)

- **Problem** — The Trading domain listed `Private DvP (link missing)` as a primary use case. The entry had no canonical use-case page.
- **Fix** — Remove the placeholder. The final diff links Private Repo, Private Stocks and Private Commodities as primary use cases and adds separate links for the Atomic DvP and Private Trade Settlement approaches.
- **Proof** — Checked the replacement paths in the merged diff. The PR records passing content validation, terminology checks and `git diff --check`. Merged on 13 October 2026 as [`8451037`](https://github.com/ethsystems/map/commit/84510370647cf02006ff758c27516e886a7f9638).

<a name="ethsystems-works-51--rotortree-snapshot-root-panic--medium-api-reliability-self-assessed"></a>

### 13. [EthSystems Works #51 — rotortree snapshot root panic](https://github.com/ethsystems/works/pull/51) · Medium API reliability (self-assessed)

- **Problem:** A valid `rotortree` Merkle tree can grow to `MAX_DEPTH`, but `TreeSnapshot` stores only `MAX_DEPTH` non-root level slots in `levels`. Calling the public `level_len(depth)` or `get_node(depth, index)` at that depth indexed `levels[MAX_DEPTH]` and panicked, including before the tree reached full leaf capacity. This is a reproducible reliability defect; a production outage or remote exploit has not been demonstrated.
- **Fix:** Return the root count and hash from the separately stored `self.root` whenever `level == self.depth`, preserving empty-tree behaviour and returning `IndexOutOfRange` for invalid indices. The patch touches only the two accessors and their tests in `crates/rotortree`.
- **Proof:** The maximum-depth regression failed with an index-out-of-bounds panic against upstream `e5061c8` (1 failing and 3 passing focused tests); applying the patch gave **4 passing, 0 failing**. Regression coverage exercises sequential and batch insertion, binary depth 1 and 3, ternary depth 2, and each maximum-depth leaf-count range, plus empty/single-leaf controls. Locally, **107 default**, **180 all-feature** and **107 no-default-feature** tests passed, alongside Clippy, nightly formatting and diff checks. Maintainer `rymnc` requested test simplification and a signed commit, approved the revised patch, and merged it on 13 October 2026 as [`1449c62`](https://github.com/ethsystems/works/commit/1449c629b33e312d56b3a2f8e4511b69878524ba).

<a name="defillama-sdk-266--log-cache-range-union"></a>

### 14. [DeFiLlama SDK #266 — log-cache range union](https://github.com/DefiLlama/defillama-sdk/pull/266)

- **Problem:** A forced nested `getLogs` fetch could reduce the recorded end of a cached range. With an existing [100, 1000] range, requesting [500, 600] wrote [100, 610], because the merge replaced `toBlock` with the shorter padded range's end.
- **Fix:** Keep `Math.max` of covered end blocks when ranges overlap; merge adjacent ranges while leaving disjoint ranges separate.
- **Proof:** A red/green Jest regression exercises nested, right-extending, adjacent and disjoint ranges. **4 tests**, the TypeScript build and diff check passed. Maintainer `g1nt0ki` approved; merged 12 October 2026 as [`f286359`](https://github.com/DefiLlama/defillama-sdk/commit/f286359540579f4f1a9f41366182f8fab3519e2e). This fixes cache bookkeeping; no missing-log incident is claimed.

<a name="whatsapp-erlang-language-platform-284--grouped-common-test-runnables"></a>

### 15. [WhatsApp Erlang Language Platform #284 — grouped Common Test runnables](https://github.com/WhatsApp/erlang-language-platform/pull/284)

- **Problem:** Rebar3 Common Test runnables emitted a bare group name after `--case` instead of the required `--group <name>`.
- **Fix:** Insert the flag only for grouped tests, leaving ungrouped runnables unchanged; add both grouped and ungrouped regressions.
- **Proof:** The new regression failed before the fix while **1,809 control tests passed**. Workspace tests passed in reported Linux and macOS OTP matrices. **Meta internal merge:** `robertoaloi` imported the contribution into internal review and merged it as [`05020fd`](https://github.com/WhatsApp/erlang-language-platform/commit/05020fda13fa6592538fb56ca2b1d463ab3e0c8f), as [confirmed by Meta's code-sync bot](https://github.com/WhatsApp/erlang-language-platform/pull/284#issuecomment-6078526521). GitHub marks the original PR closed rather than natively merged. The [14 October 2026 release](https://github.com/WhatsApp/erlang-language-platform/releases/tag/2026-10-14) credits the fix.

<a name="nethermind-juno-4146--json-rpc-notification-responses"></a>

### 16. [Nethermind Juno #4146 — JSON-RPC notification responses](https://github.com/NethermindEth/juno/pull/4146)

- **Problem:** Juno sent JSON-RPC error responses to requests without an `id` when the method was unknown or parameters invalid. The JSON-RPC specification prohibits responding to notifications.
- **Fix:** Suppress all notification responses after dispatch without suppressing error responses for ID-bearing requests.
- **Proof:** Four red/green regressions cover unknown-method notifications, invalid-parameter notifications, notification-only batches and mixed batches. The relevant packages passed `go test -race`. Approved and merged 8 October 2026 as [`4c6b4d3`](https://github.com/NethermindEth/juno/commit/4c6b4d318a6eb038fabb918db7cf271763827c80). Shipped in [Juno v0.17.0](https://github.com/NethermindEth/juno/releases/tag/v0.17.0), whose release notes credit this PR as my first contribution.

<a name="polygon-agent-cli-142--x402-dry-run-transaction-safety"></a>

### 17. [Polygon Agent CLI #142 — x402 dry-run transaction safety](https://github.com/0xPolygon/polygon-agent-cli/pull/142)

- **Problem:** `x402-pay` unconditionally broadcast a funding transaction, bypassing the CLI's persisted dry-run transaction mode.
- **Fix:** Use the existing transaction-mode resolver and shared `--broadcast` / `--dry-run` flags; return a structured payment preview rather than moving funds while dry-run is active.
- **Proof:** Regression coverage checks standard x402 and Bazaar payment paths do not submit transactions in dry-run mode. [Fixes issue #130](https://github.com/0xPolygon/polygon-agent-cli/issues/130); merged 8 October 2026 as [`b3e81d0`](https://github.com/0xPolygon/polygon-agent-cli/commit/b3e81d0fa64b95a5c83f0a1073ee9ea30ec2b6d8).

<a name="solana-foundation-website-2150--transaction-v1-slot-time-blocks"></a>

### 18. [Solana Foundation website #2150 — transaction v1 slot-time blocks](https://github.com/solana-foundation/solana-com/pull/2150)

- **Problem:** The slot-time endpoint requested full blocks with `maxSupportedTransactionVersion: 0`; blocks containing v1 transactions could be rejected, leaving a stale cached block instead of a fresh update.
- **Fix:** Permit version 1 in the RPC request and preserve program-ID resolution using static and loaded address-table keys.
- **Proof:** Regressions assert the exact JSON-RPC request and address resolution; **3 Vitest tests**, TypeScript and formatting checks passed. [Fixes #2141](https://github.com/solana-foundation/solana-com/issues/2141); merged as [`a233795`](https://github.com/solana-foundation/solana-com/commit/a233795827fb9ff3581a3f915c7ccc518c1a7b88).

<a name="promptfoo-mcp-agent-provider-95--measured-model-usage"></a>

### 19. [Promptfoo MCP Agent Provider #95 — measured model usage](https://github.com/promptfoo/mcp-agent-provider/pull/95)

- **Problem:** The provider estimated OpenAI token counts from message counts and fabricated costs with a hard-coded price, producing misleading usage reports across multi-step tool calls.
- **Fix:** Aggregate actual API-reported token usage across ReAct iterations, omit incomplete totals, and remove unsupported synthetic costs.
- **Proof:** Real-SDK loopback regressions cover measured, zero and missing usage; the reported **31 tests** and Node 20/22/24 checks passed. Merged upstream as [`6478386`](https://github.com/promptfoo/mcp-agent-provider/commit/6478386ffb7207ab832d0e0849695213e09bef71).

<a name="promptfoo-mcp-agent-provider-96--legacy-sse-fallback"></a>

### 20. [Promptfoo MCP Agent Provider #96 — legacy SSE fallback](https://github.com/promptfoo/mcp-agent-provider/pull/96)

- **Problem:** The advertised Streamable HTTP → SSE fallback ran only during transport construction, but incompatibility with an SSE-only endpoint surfaces during `client.connect()`.
- **Fix:** On qualifying 4xx `StreamableHTTPError` negotiation failures, close the failed connection, create a fresh MCP client and retry with legacy SSE; do not mask unrelated transport failures.
- **Proof:** An end-to-end legacy SSE server fixture validates the negotiated fallback, with Node 20/22/24 and Biome CI reported passing. Merged as [`bf5f668`](https://github.com/promptfoo/mcp-agent-provider/commit/bf5f6686bb62c16c73b13bf0220c7205263b66bf).

<a name="openai-codex-security-1306--vyper-source-scan-coverage"></a>

### 21. [OpenAI Codex Security #1306 — Vyper source scan coverage](https://github.com/openai/codex-security/pull/1306)

- **Problem:** The shared source-code allowlist omitted `.vy` files, so Vyper-only changes could disappear before scan inventory and ranking.
- **Fix:** Include Vyper sources in existing text-code inventory and ranking flows and add targeted regressions for repository, revision-diff and local-patch paths.
- **Proof:** An OpenAI maintainer independently reproduced **five failing inventory/ranking cases on the parent** and verified that all passed after the fix. **295 tests** passed across affected modules, with no blocking behaviour issues. Merged 5 October 2026 as [`d327164`](https://github.com/openai/codex-security/commit/d32716411f3d7b87085b36f9de55f0ac40454c42).

<a name="nethermind-14228--jwt-cache-expiration"></a>

### 22. [Nethermind #14228 — JWT cache expiration](https://github.com/NethermindEth/nethermind/pull/14228)

- **Problem:** A correctly signed JWT that initially passed validation could continue to authenticate from the cache after its explicit `exp`, because cache hits used the `iat` window without rechecking expiration.
- **Fix:** Apply the same exclusive expiry boundary (`now >= exp` is rejected) to cached and fresh validation while retaining the normal valid-token fast path.
- **Proof:** A deterministic clock-based reproduction compares fresh validation against cache reuse at `iat + 1`, exact expiry, and subsequent times. Both manual HS256 and library fallback paths are covered by regressions. Merged 5 October 2026 as [`eeca264`](https://github.com/NethermindEth/nethermind/commit/eeca26403bef8420a980ebde700c9e338caa75e8).


<a name="ethereumorg-19368--contributor-onboarding-in-the-readme"></a>

### 23. [ethereum.org #19368 — contributor onboarding in the README](https://github.com/ethereum/ethereum-org-website/pull/19368)

- **Problem** — The contributor walkthrough had two stale links. It also pointed locale configuration at `.env` after creating `.env.local` and skipped the staging and upstream-tracking steps needed by a new contributor.
- **Fix** — Link the current Translation Program and emoji-key pages, use `.env.local` for build locales, and document staging and first-push tracking.
- **Proof** — The PR records working replacement links, locale selection with the pinned Next.js loader, and local Git fixtures that reproduced the failed commit and first push. The change was listed in the [v11.29.1 deployment](https://github.com/ethereum/ethereum-org-website/pull/19430). Merged on 7 October 2026 as [`efb29af`](https://github.com/ethereum/ethereum-org-website/commit/efb29af97f47ab076c52bcb3771dd5facafdc377).

<a name="nethermind-net-risc-v-18--stateless-executor-source-link"></a>

### 24. [Nethermind .NET RISC-V #18 — Stateless Executor source link](https://github.com/NethermindEth/dotnet-riscv/pull/18)

- **Problem** — The README pointed to a deleted feature branch, so readers could not reach the Stateless Executor source that the repository builds for RISC-V.
- **Fix** — Point the link to [`Nethermind.Stateless.Executor` on `master`](https://github.com/NethermindEth/nethermind/tree/master/src/Nethermind/Nethermind.Stateless.Executor).
- **Proof** — The PR records a 404 for the old ref and verifies that the replacement directory contains the project and implementation. The merged diff changes one README line. Merged on 7 October 2026 as [`41038a0`](https://github.com/NethermindEth/dotnet-riscv/commit/41038a063eecd9c6998917c7be0b1170c08d829d).

<a name="nethermind-pluto-714--topic-labels-at-the-default-log-level"></a>

### 25. [Nethermind Pluto #714 — topic labels at the default log level](https://github.com/NethermindEth/pluto/pull/714)

- **Problem** — At `info`, a global tracing filter disabled DEBUG topic spans before the metrics layer could see them. Warning and error counters could therefore carry an empty `topic` label.
- **Fix** — Give metrics a separate filter for topic spans and warning/error events, keeping unrelated DEBUG callsites disabled. The maintainer's final PR limits the counters to Pluto events.
- **Proof** — The regression uses the real tracing initialiser and checks labelled warning and error counters at `info`. A control verifies that unrelated DEBUG spans stay disabled. The original PR reports **15 passing tracing tests**, and the upstream PR retains the fix and regression commits. Landed through [#720](https://github.com/NethermindEth/pluto/pull/720) on 6 October 2026 as [`0aec20b`](https://github.com/NethermindEth/pluto/commit/0aec20b0cb62ad1d02d25553da9f423dd40cd1be).

<a name="blackrock-hola-69--forrester-quick-start-benchmark"></a>

### 26. [BlackRock HOLA #69 — Forrester quick-start benchmark](https://github.com/blackrock/HOLA/pull/69)

- **Problem** — Both quick-start examples used `sin(term / 2)` where the benchmark uses `sin(2 * term)`, with `term = 6 * x - 2`. Copying either example gave a different objective from the one the stated minimum describes.
- **Fix** — Correct both examples and update the README's approximate minimum to `-6.02074`.
- **Proof** — A regression extracts the two documented functions and compares each with the repository benchmark at five inputs. All **10 comparisons** failed before the correction and agreed within `1e-12` after it. Merged on 6 October 2026 as [`55647c9`](https://github.com/blackrock/HOLA/commit/55647c9350fc235a39663820a3c9e12027246dfe).

<a name="ethsystems-web-46--sibling-rfp-routes"></a>

### 27. [EthSystems Web #46 — sibling RFP routes](https://github.com/ethsystems/web/pull/46)

- **Problem** — Bare links between RFP files fell through to `/use-cases/` because RFPs are outside the graph and the relative filename had no directory prefix.
- **Fix** — Route bare `rfp-*.md` links to `/rfps/` before graph matching and preserve query strings and fragments.
- **Proof** — Checked all seven current sibling-link occurrences. Regressions cover four RFP targets and fragment preservation. The combined upstream change reports **62/62** passing tests. Grouped in [#49](https://github.com/ethsystems/web/pull/49) and cherry-picked to `main` on 5 October 2026 as [`3330621`](https://github.com/ethsystems/web/commit/3330621002207df1135787d66eb93c25792c773b).

<a name="ethsystems-web-47--glossary-definitions-on-the-next-line"></a>

### 28. [EthSystems Web #47 — glossary definitions on the next line](https://github.com/ethsystems/web/pull/47)

- **Problem** — The glossary parser read only the term line. The Data Availability Layer entry put its definition on the next line, so the generated definition was empty and the term was omitted from the public glossary.
- **Fix** — Use the immediate continuation line when the inline definition is empty. Keep category headings and new terms out of the previous definition.
- **Proof** — Tests cover the missing definition, the next-term and next-category boundaries, and existing inline and linked terms. The regenerated entry contains its definition, and the combined upstream change reports **62/62** passing tests. Grouped in [#49](https://github.com/ethsystems/web/pull/49) and cherry-picked to `main` on 5 October 2026 as [`3330621`](https://github.com/ethsystems/web/commit/3330621002207df1135787d66eb93c25792c773b).

<a name="ethsystems-web-48--private-bonds-link-to-the-shielding-pattern"></a>

### 29. [EthSystems Web #48 — Private Bonds link to the Shielding pattern](https://github.com/ethsystems/web/pull/48)

- **Problem** — The Private Bonds Part 3 write-up linked to a file removed when [Map #112](https://github.com/ethsystems/map/pull/112) consolidated the shielded balances material.
- **Fix** — Point `Custom UTXO` to the Shielding pattern and add the matching `map_refs` entry.
- **Proof** — Checked the current Map tree, parsed the frontmatter and inspected the generated HTML. The combined upstream change reports **62/62** passing tests. Grouped in [#49](https://github.com/ethsystems/web/pull/49) and cherry-picked to `main` on 5 October 2026 as [`3330621`](https://github.com/ethsystems/web/commit/3330621002207df1135787d66eb93c25792c773b).

<a name="promptfoo-modelaudit-1859--safetensors-files-misrouted-as-zlib"></a>

### 30. [Promptfoo ModelAudit #1859 — SafeTensors files misrouted as zlib](https://github.com/promptfoo/modelaudit/pull/1859)

- **Problem** — A valid SafeTensors header length such as 8,312 bytes begins with a prefix that also looks like preset-dictionary zlib. Compression took precedence, sending a valid model shard to the wrong scanner.
- **Fix** — Use the SafeTensors route when the extension is native and the bounded header fully parses. Ambiguous or inconclusive inputs and genuine zlib fixtures retain compression routing.
- **Proof** — The upstream PR reports **19 passing focused cases**, including all five reported header sizes, genuine compressed input and scanner coverage for benign and malicious metadata. The installed SafeTensors reader accepted all five fixtures. [Fixes #1854](https://github.com/promptfoo/modelaudit/issues/1854). Landed through [#1863](https://github.com/promptfoo/modelaudit/pull/1863) on 4 October 2026 as [`2a5185a`](https://github.com/promptfoo/modelaudit/commit/2a5185a32a3993ac6188b20a04fe6cec7cc2eddc). Released in [v0.2.54](https://github.com/promptfoo/modelaudit/releases/tag/v0.2.54).

<a name="centrifuge-api-v3-489"></a>

### 31. [Centrifuge API v3 #489](https://github.com/centrifuge/api-v3/pull/489)

- **Problem:** Pharos (chain ID `1672`) was listed in `networkNames` but missing from `explorerUrls`. The indexer therefore initialised `Blockchain.explorer` without a Pharos URL, causing Token and NAV links to fall back to the pool dashboard.
- **Fix:** Added `https://www.pharosscan.xyz` for chain `1672` in `src/chains.ts`, preserving the existing chain and RPC configuration.
- **Proof:** Checked the chain mapping and the timekeeper path that populates `Blockchain.explorer`. The merged diff adds only the explorer mapping. A maintainer approved the PR and merged it on 1 October 2026, closing [#479](https://github.com/centrifuge/api-v3/issues/479). Existing Pharos rows may still require a refresh or reindex because `getOrInit` does not update initialised records.

<a name="ethereumorg-19272"></a>

### 32. [ethereum.org #19272](https://github.com/ethereum/ethereum-org-website/pull/19272)

- **Problem:** The Glamsterdam roadmap's manually maintained proposal notice had drifted from the generated upgrade data. It still listed EIP-7610 and omitted the scheduled networking proposals EIP-8070, EIP-8136 and EIP-8189.
- **Fix:** Removed EIP-7610 and added eth/72 (EIP-8070), EIP-8136 and snap/2 (EIP-8189) to the English source. The change preserves the page's structure and leaves translation propagation to the internationalisation pipeline.
- **Proof:** Checked the proposal list against `src/data/upgrades/generated.ts`. The PR records zero markdownlint errors and a passing `git diff --check`; its diff changes only `public/content/roadmap/glamsterdam/index.md`. A maintainer approved and merged the PR on 1 October 2026. The team also added me as an ethereum.org maintenance contributor in [#19364](https://github.com/ethereum/ethereum-org-website/pull/19364).

<a name="google-chrome-platform-status-6946"></a>

### 33. [Google Chrome Platform Status #6946](https://github.com/GoogleChrome/chromium-dashboard/pull/6946)

- **Problem:** Chrome Platform Status hides stage-level actions when an API Owners gate is present and expects the gate column to render actions associated with that gate. The experiment-extension email action had an empty gate-type list in the normal launch, fast-track and deprecation process definitions, so neither rendering path displayed it.
- **Fix:** Associated all three email actions with `GATE_API_EXTEND_ORIGIN_TRIAL` and added one regression covering the three process definitions. The change leaves the email URL, prerequisites and approval rules untouched.
- **Proof:** The regression failed on the unpatched `main` branch for all three definitions with `AssertionError: 3 not found in []`, then passed after the fix. Ruff lint and formatting, the licence check, `git diff --check` and mypy across 363 source files also passed. A maintainer approved the change, and Google's merge queue landed it as [`ca80064`](https://github.com/GoogleChrome/chromium-dashboard/commit/ca80064d966c5f58df7c25941ebe8416d7ae9a03) on 29 September 2026. It was subsequently included in the closed [deployment release #6950](https://github.com/GoogleChrome/chromium-dashboard/issues/6950).

<a name="ethereum-eips-12360--eip-8141-calldata-floor-dependency"></a>

### 34. [Ethereum EIPs #12360 — EIP-8141 calldata floor dependency](https://github.com/ethereum/EIPs/pull/12360)

- **Problem** — EIP-8141 used EIP-7976's calldata-gas constants in its intrinsic-gas calculation while omitting EIP-7976 from its required dependencies.
- **Fix** — I proposed adding EIP-7976 to `requires`. The maintainer adopted that correction in a replacement PR, removed the superseded EIP-7623 dependency and updated the calldata-floor references in the body.
- **Proof** — The original PR records Markdown lint and dependency-order checks. The maintainer-authored [#12395](https://github.com/ethereum/EIPs/pull/12395) superseded my PR and merged on 29 September 2026 as [`5c0236f`](https://github.com/ethereum/EIPs/commit/5c0236f9c216640909aad63d70ec9df59ac85457).

<a name="ethsystems-web-45--glossary-link-preservation"></a>

### 35. [EthSystems Web #45 — glossary link preservation](https://github.com/ethsystems/web/pull/45)

- **Problem** — Glossary generation stripped URLs from linked term names, and definitions could render links as literal Markdown.
- **Fix** — Retain link metadata in the generated glossary and render definition links. Resolve internal Map links to site routes and root Map documents to their GitHub source.
- **Proof** — The upstream commit records **58/58** passing tests and a **218-page** build. Regressions cover linked terms, external references and internal routes. Cherry-picked upstream on 28 September 2026 as [`63f99bd`](https://github.com/ethsystems/web/commit/63f99bde89cddf512f0304b9912fd25e8917ca81).

<a name="openai-codex-security-1020"></a>

### 36. [OpenAI Codex Security #1020](https://github.com/openai/codex-security/pull/1020)

- **Problem:** The `scan --patch` workflow could report success when the model returned `no_change` without explaining how it had checked the finding. A claim that no fix was needed could therefore clear the finding without verification evidence.
- **Fix:** Extended the existing verification requirement to `no_change` results. Missing or whitespace-only verification now produces a failed result and exit code 2. Valid `no_change` results remain accepted, and the response instructions explicitly require verification for both successful statuses. This checks that an explanation exists; it does not establish that the explanation is true.
- **Proof:** Controlled before/after tests reproduced success for missing and blank verification on the unpatched code and rejection after the fix. The [maintainer confirmed](https://github.com/openai/codex-security/pull/1020#issuecomment-5855914520) 72 focused tests passed, plus two full SDK runs with 3,242 passed, 50 skipped and no failures each. Builds, type checks, formatting and GitHub CI passed. Approved and merged on 27 September 2026, with the original fix credited to me.

<a name="openai-codex-security-1021--solidity-diff-scan-coverage"></a>

### 37. [OpenAI Codex Security #1021 — Solidity diff scan coverage](https://github.com/openai/codex-security/pull/1021)

- **Problem** — The shared text-code allowlist omitted `.sol`. A Solidity-only diff could produce zero in-scope files and no review items. Solidity sources also dropped out of ranking inputs.
- **Fix** — Add `.sol` to the allowlist and use the existing sampled-text preview. Full-repository inventory already included these files.
- **Proof** — Five inventory and ranking regressions failed on `main` and passed with the fix. Coverage spans repository ranking, revision diffs and local patches. The PR reports **1,151 passing plugin tests**, with 8 skipped, alongside the SDK build and compatibility checks. Merged on 25 September 2026 as [`ce7f75c`](https://github.com/openai/codex-security/commit/ce7f75c1efd5c483bd4500bda9175775bd993daa).

<a name="ethsystems-web-44--broken-proof-of-concept-links"></a>

### 38. [EthSystems Web #44 — broken proof-of-concept links](https://github.com/ethsystems/web/pull/44)

- **Problem** — Four write-ups linked to the PoC repository's nonexistent `main` branch or an obsolete private-trade-settlement path.
- **Fix** — Use the PoC repository's `master` branch and correct the cross-chain spec directory.
- **Proof** — The upstream commit records checks that all three replacement spec paths exist. The [maintainer confirmed the cherry-pick](https://github.com/ethsystems/web/pull/44#issuecomment-5831969353). Cherry-picked upstream on 24 September 2026 as [`82bd4f0`](https://github.com/ethsystems/web/commit/82bd4f01ebe61dc1fec30d0b1c41de0c67ae1019).

<a name="openai-guardrails-js-148"></a>

### 39. [OpenAI Guardrails JS #148](https://github.com/openai/openai-guardrails-js/pull/148)

- **Problem:** The vector-store helper checked a path's extension before checking whether the path was a file or directory. Directories such as `documents.v1` or `archive.zip` were therefore rejected even when they contained supported documents.
- **Fix:** Classified the path first, then applied the extension allowlist only to regular files. Directory filtering remains case-insensitive and non-recursive.
- **Proof:** Three dotted-directory regressions failed against the original implementation and passed after the fix. All 882 tests, build, lint and documentation checks passed locally; CI passed on Node.js 22, 24 and 26, together with both CodeQL analyses. OpenAI approved and merged the PR on 24 September 2026.

<a name="promptfoo-11243"></a>

### 40. [Promptfoo #11243](https://github.com/promptfoo/promptfoo/pull/11243)

- **Problem:** The Google Vertex Claude provider resolved `top_p` and `top_k` aliases with logical OR. Because `0` is falsy, an explicit zero could be replaced by the camelCase alias or omitted, changing the sampling configuration sent to the model.
- **Fix:** Replaced the two falsy fallbacks with nullish coalescing so only a missing value falls through, while preserving the existing alias precedence and newer-model sampling suppression.
- **Proof:** Regression cases cover explicit zero values against non-zero aliases. The maintainer independently verified that reverting either fallback makes its regression fail, then ran 1,146 Google provider tests, typecheck, the CLI build, formatting/lint and ten mocked-transport QA configurations. All 57 applicable PR checks were green before merge on 29 September 2026.

<a name="go-ethereum-35710"></a>

### 41. [go-ethereum #35710](https://github.com/ethereum/go-ethereum/pull/35710)

- **Problem:** Ethereum JSON-RPC documentation no longer matched the implementation.
- **Fix:** Corrected block selectors, exclusions and response examples.
- **Proof:** Checked the documentation against the current RPC source.

<a name="google-skill-reach-23"></a>

### 42. [Google Skill Reach #23](https://github.com/google/skill-reach/pull/23)

- **Problem:** CSV and JSONL round trips dropped query metadata and could change evaluation results.
- **Fix:** Preserved acceptable skills, notes and custom columns across both formats.
- **Proof:** Added round-trip coverage and passed the full test, type, lint and documentation checks.

<a name="nasa-delta-158"></a>

### 43. [NASA Delta #158](https://github.com/nasa/delta/pull/158)

- **Problem:** Cache eviction tried to remove regular files as directories, and the text-file exclusion used the wrong suffix.
- **Fix:** Removed files and directories with the correct operation and repaired the `.txt` check.
- **Proof:** Regression tests cover files, directories and protected CSV/TXT files.

<a name="samsung-credsweeper-951"></a>

### 44. [Samsung CredSweeper #951](https://github.com/Samsung/CredSweeper/pull/951)

- **Problem:** CRX3 browser extensions were read with the older CRX2 header layout, so scans could start at the wrong ZIP offset.
- **Fix:** Added version-specific CRX2 and CRX3 parsing and rejected unsupported or truncated headers.
- **Proof:** Focused and scanner regression suites passed with lint and type checks.

<a name="ethsystems-map-200"></a>

### 45. [EthSystems Map #200](https://github.com/ethsystems/map/pull/200)

- **Problem:** The ERC-3643 guide treated investor transfers, minting and administrative transfers as if they followed the same checks.
- **Fix:** Separated the transfer paths and clarified owner, agent, compliance and censorship assumptions.
- **Proof:** Compared the text with the canonical specification and reference implementation.

<a name="consensys-ask-o11y-226"></a>

### 46. [Consensys Ask O11y #226](https://github.com/Consensys/ask-o11y-plugin/pull/226)

- **Problem:** Newer OpenAI models rejected the legacy `max_tokens` request field.
- **Fix:** Replaced it with `max_completion_tokens` throughout request construction and diagnostics.
- **Proof:** A regression test requires the new field and rejects the old one.

<a name="nethermind-13747"></a>

### 47. [Nethermind #13747](https://github.com/NethermindEth/nethermind/pull/13747)

- **Problem:** Engine API V3 and later accepted explicit null values for required payload fields and returned the wrong class of error later.
- **Fix:** Return invalid parameters for explicit `null` in `withdrawals`, `blobGasUsed` or `excessBlobGas`, and for omitted `withdrawals`.
- **Proof:** Regressions cover explicit `null` and omitted fields. The baseline already rejected omitted `blobGasUsed` and `excessBlobGas`. The maintainer PR superseded my [#13734](https://github.com/NethermindEth/nethermind/pull/13734) while retaining my commits and authorship.

<a name="nethermind-13755"></a>

### 48. [Nethermind #13755](https://github.com/NethermindEth/nethermind/pull/13755)

- **Problem:** Nethermind still served `engine_getPayloadV5` at Amsterdam, even though its V5 response cannot carry Amsterdam's `slotNumber` or `blockAccessList`. It could return an outdated payload shape instead of `-38005 Unsupported fork`.
- **Fix:** Limited V5 to its Osaka window and rejected it once block-level access lists are enabled.
- **Proof:** The focused regression failed in both fixture modes before the fix and passed 2/2 afterward. The V4/V5/V6 fork-boundary suite passed 33/33, preserving Osaka V5 behaviour. The PR received three maintainer approvals, merged into `master` on 24 September 2026 and closed [#13713](https://github.com/NethermindEth/nethermind/issues/13713) as completed.

<a name="defillama-pegged-assets-927"></a>

### 49. [DefiLlama Pegged Assets #927](https://github.com/DefiLlama/peggedassets-server/pull/927)

- **Problem:** The EURR description named Revolut as the issuer, although Bridge Building S.A. issues the token and Revolut distributes it.
- **Fix:** Corrected the issuer and distributor attribution.
- **Proof:** Cross-checked the wording against the issuer and distributor sources. No supply or pricing logic changed.

<a name="dedaub-srcup-action-1"></a>

### 50. [Dedaub srcup action #1](https://github.com/Dedaub/srcup-ci-action/pull/1)

- **Problem:** The workflow example used stale action versions and unsafe output handling.
- **Fix:** Updated the versions, explained the input/output key difference and used safe report printing.
- **Proof:** Checked the example against the published action interface.

<a name="dedaub-source-warnings-action-1"></a>

### 51. [Dedaub source warnings action #1](https://github.com/Dedaub/srcwarnings-ci-action/pull/1)

- **Problem:** The documentation used underscore input names that the action does not accept.
- **Fix:** Documented the correct hyphenated inputs, required API key and supported options.
- **Proof:** Checked both examples against the action definition.

<a name="solana-web3js-3943"></a>

### 52. [Solana web3.js #3943](https://github.com/solana-foundation/solana-web3.js/pull/3943)

- **Problem:** Smoke tests did not execute IIFE browser bundles, so an unreplaced `process.env["NODE_ENV"]` reference could ship unnoticed.
- **Fix:** Corrected the replacement and made smoke tests execute both IIFE bundles.
- **Proof:** Production builds and CJS, ESM, browser, native and IIFE smoke tests passed.

<a name="trail-of-bits-skills-306"></a>

### 53. [Trail of Bits Skills #306](https://github.com/trailofbits/skills/pull/306)

- **Problem:** An obsolete configuration invoked the removed `codex mcp-server` command and could break Claude Code start-up.
- **Fix:** I proposed removing the obsolete server registration in [#303](https://github.com/trailofbits/skills/pull/303) after `selimacerbas` reported the start-up failure in [#301](https://github.com/trailofbits/skills/issues/301). The maintainer shipped the removal in the broader #306 clean-up.
- **Proof:** Plugin loadability, metadata, shell, Bats, JavaScript and temporary Git fixture checks covered the change.
