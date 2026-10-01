# Bonding Curve Security Primer

A reference for auditing bonding-curve contracts: what a bonding curve is as an attack
surface, the invariants it must hold, a catalogue of known attack classes with
mitigations, an audit checklist, and worked examples. Compiled while auditing the Alt
Fun launchpad (`Bonding.sol` / `Router.sol` / `Pair.sol` — a constant-product curve
hybridised against an external leveraged-token price); see `report.md` in this repo for
the full worked audit trail this primer is grounded in.

---

## 1. What makes a bonding curve a distinct attack surface

A bonding curve encodes a protocol's pricing *and* its economic incentives directly
into on-chain math — there is no order book, no external market maker, and often no
oracle. That makes it:

- **Self-contained**: every price the curve ever quotes is a pure function of its own
  stored state (reserves, a constant, a formula) — nothing external can move the price
  unless the curve design explicitly lets it in (e.g. an oracle-hybrid curve, see §6).
- **Incentive-aligned by construction, but gameable**: the formula *is* the economics.
  A bug in the formula, or a gap between the formula's assumptions and how it's
  actually called, is directly an economic exploit — there's no separate "business
  logic" layer to catch it.
- **State-transition-heavy**: most interesting bonding curves aren't just "buy/sell
  forever" — they have a lifecycle (seeding → trading → a graduation/migration event
  → post-graduation venue). Each transition is a place invariants can silently break.

Treat the curve's formula and its lifecycle transitions as the primary attack surface,
not just the individual `buy()`/`sell()` entry points.

---

## 2. Core invariants

These should hold for *any* bonding curve; verify each one explicitly, don't assume it
from the formula alone. §2.1 is the essential short list; §2.2 onward is an extended,
categorized catalogue for when you want to work through a design exhaustively rather
than selectively.

### 2.1 The essential eight

| # | Invariant | Why it breaks in practice |
|---|---|---|
| 1 | **The pricing constant (`k`, or equivalent) never changes after it's set**, except through an explicitly intended re-seed. | A mutable constant turns every holder's expected payout into whatever the mutator wants. |
| 2 | **No round trip (buy then sell, or sell then buy, of the same size) is profitable** for the trader, beyond what fees refund. | Usually broken by inconsistent rounding direction between the buy and sell formulas, or by one side reading *stored* reserves and the other reading *live* balances. |
| 3 | **The curve's real token balance always equals what it owes sellers** — i.e. `realBalance(curve) == storedReserve - virtualOffset` (or the design's equivalent). | Donations, external pre-seeding, or a formula that derives payout from `balanceOf()` instead of stored state, break this independently in each direction. |
| 4 | **Every output is capped by the real balance the curve actually holds** — a quote can never promise more than exists. | Curves that compute the "ideal" constant-product output without clamping to `tokenBalance()` will attempt to pay out virtual supply that was never real. |
| 5 | **Price moves monotonically with net flow**: a buy never lowers price, a sell never raises it, for any input size including the smallest and largest reachable. | Breaks at rounding boundaries, at the "last unit of supply" edge, or across a lifecycle transition (price silently resets instead of continuing). |
| 6 | **Zero and dust inputs either revert or are no-ops** — they never produce a non-zero state change for a zero-ish input, and never divide by a quantity that can be zero. | Classic in hand-rolled curve formulas with no `amount > 0` guard, or a denominator built from user-influenced totals. |
| 7 | **The lifecycle only moves forward** (e.g. `Seeding → Trading → Graduating → Graduated`), and every state-dependent check reads the *current* state, not a cached one. | A check written against an assumption that's silently stopped holding once a transition fires is a common post-graduation (or post-pause, post-upgrade) bug. |
| 8 | **A migration/graduation event opens the destination venue at the curve's own closing price — zero gap.** If anyone can pre-seed that venue before migration, their pre-seed can't let them claim more value than they put in. | The single highest-value invariant to break in practice: a non-zero gap between curve-close price and destination-open price is a direct, quantifiable extraction. |

### 2.2 Pricing & curve math

- Marginal price is strictly positive and finite for every reachable state, including right at the supply ceiling.
- Output is monotonic in input: a larger input never yields a smaller output than a smaller input, at the same state.
- The inverse formula (if one exists — "what input gives this output") is consistent with the forward formula to within its own documented tolerance.
- No discontinuity in price exists at a lifecycle boundary under the design's own definition of "equal."
- Splitting one large trade into many small ones produces, within rounding, the same cumulative output as the single trade — no splitting arbitrage.
- A calculus-derived bulk-pricing shortcut matches a step-wise numerical integration to within tolerance, if one is used.
- No input produces an output exceeding the theoretical maximum (total supply, real reserve held).
- No unsigned computation silently underflows into a huge value instead of reverting, checked explicitly rather than trusted to the compiler's revert-on-underflow alone.
- Virtual reserves (shaping initial depth) can never be confused with, or allow a drain of, real reserves.
- The curve's price sensitivity to trade size stays within whatever range the protocol's own documentation claims, across the full supply range.
- A quoting/preview function (`getAmountOut`-equivalent) returns exactly what the corresponding state-changing call would produce, for the same state and input.
- Two mathematically equivalent ways of expressing the same formula (e.g. `a*b/c` vs. `a/c*b`) are checked for producing identical on-chain results — they don't, and the audit should know which one is actually implemented.

### 2.3 Conservation & balance accounting

- `sum(all tracked user balances/shares) == the protocol's own internal total-supply counter`, at all times.
- Every unit of reserve asset entering via a tracked entry point is accounted for in exactly one place — no double-count, no silent drop.
- `assets in == assets out + fees retained + assets still escrowed`, exactly, after every single state-changing call.
- No function increases the protocol's own liability without a matching increase in backing assets.
- A direct donation (bypassing the normal entry point) never increases any individual user's claimable balance.
- The contract's own balance of every asset it holds is always ≥ the sum of everything it owes out in that asset.
- Fee-accrual ledgers (per-recipient claimable balances) sum exactly to the global "total accrued" counter.
- A claim/withdraw zeroes or correctly decrements the claimant's balance in the same transaction it pays out — no repeat claims.
- Internal accounting units (shares, virtual balances) convert back to the real asset consistently: minting shares for X then redeeming returns ≤ X, never more.
- Sweeping "excess"/donated funds never sweeps funds actually owed to someone.
- A partial failure mid-multi-step operation (one leg reverts) leaves no asset half-moved — the whole operation is atomic.

### 2.4 Reserve state vs. real balance

- `IERC20(asset).balanceOf(curve) >= storedReserve - virtualOffset`, always, including immediately after an external donation.
- A token donated directly to the curve (bypassing `buy()`) can be swept, burned, or left idle, but never double-spent as if raised through a real trade.
- Cached/stored reserve numbers are never permanently desynchronized from reality in a way a future trade can exploit for a more favourable price.
- A `sync()`-style function that reconciles stored state to the live balance can never be used to manufacture reserves the curve doesn't actually hold.
- A 1-wei griefing deposit cannot flip a branch of the pricing logic that wasn't meant to bind (e.g. a "real balance" cap check).
- Skimming excess balance (if supported) never skims balance that's actually backing an outstanding obligation.

### 2.5 Zero, dust & boundary values

- `buy(0)` / `sell(0)` either revert or are provably no-ops on every piece of state.
- The smallest non-zero trade (1 wei of input) either reverts cleanly or produces a bounded, non-exploitable output.
- A trade sized exactly at a threshold boundary resolves deterministically, independent of transaction ordering within the block.
- `amountIn == type(uint256).max` (or the asset's actual max transferable amount) reverts cleanly before any state mutation.
- The first trade on a freshly initialized curve behaves exactly as the formula predicts — no silent "cold start" special case.
- The last unit of sellable/buyable supply is reachable and behaves per the formula, with no off-by-one that strands or double-counts it.
- A fractional-wei output rounds in a single, documented direction, consistently across every call site, not ad hoc.
- A trade that would produce exactly zero output (due to rounding, not due to zero input) is handled explicitly — reverted, not silently accepted as a real trade.

### 2.6 Rounding & precision

- Every division is preceded by its corresponding multiplication.
- Every rounding decision favours the protocol/counterparty, never the caller, identically on the buy side and the sell side.
- Fee computations round in the protocol's favour at the smallest unit — never round a nonzero fee down to zero.
- Scaling between differing decimal precisions is applied exactly once per value — never twice, never zero times.
- A value already expressed in the target precision is never re-scaled as if still in the source precision.
- Intermediate computations use a wide enough integer type (or explicit wide-math) that the final division doesn't amplify an earlier truncation.
- Repeated application of a protocol-favourable rounding, across many calls, cannot be inverted by the caller into a caller-favourable cumulative result.
- Mixed-precision inputs (e.g. a 6-decimal and an 18-decimal token in the same formula) are tested with real mismatched values, not same-decimal stand-ins.

### 2.7 Lifecycle & state machine

- Every state transition is one-directional along the documented path; no transition, even via a bug, moves the lifecycle backward.
- A function restricted to state A reverts cleanly — never partially executes — when called in state B.
- No two mutually exclusive lifecycle states can simultaneously read `true` from their respective view functions.
- A threshold-triggered transition is evaluated against live, current state — never a value cached before the triggering call.
- A transition that's a precondition for another function is enforced inside that function itself, not only assumed from caller context.
- Re-entering a state-transition function while already mid-transition is blocked (`nonReentrant` or equivalent).
- No reachable "stuck" state exists from which no transition — including an emergency/admin one — is possible.
- A state read via a cached/memoized value anywhere in the same transaction a transition fires is proven consistent with the post-transition state.

### 2.8 Migration / graduation / venue transition

- The destination venue, once seeded, reflects the exact closing price of the curve (zero gap), for every pre-seeding shape an attacker can construct.
- Nobody can claim a share of migrated liquidity larger than what they deposited into the destination venue, valued at the curve's own closing price.
- The migration step is either fully atomic, or every intermediate phase (if split across transactions) is safe to leave indefinitely with no forced-completion window.
- A permissionless "finalize migration" step can't be front-run, griefed, or pre-empted in a way that changes who benefits.
- Funds escrowed for one identifier's migration are never reachable by a different identifier's migration logic, even sharing an underlying asset.
- Multiple alternative triggering conditions for the same migration produce economically equivalent outcomes — neither is a strictly cheaper path to the same destination price.
- Curve-side dust/remainder is correctly disposed of (burned, returned, accounted); nothing is silently stranded in the now-dead curve contract.
- The destination venue's own invariants (if it's a standard AMM) are re-verified post-seed, not assumed inherited from the curve's correctness.

### 2.9 Fees, taxes & spread

- Every entry point that moves the core asset applies the fee; no alternate reachable path moves it fee-free for an ordinary caller.
- The fee rate is bounded by a hard ceiling that can never be set above the documented maximum.
- Fee splits between recipients sum to exactly the total fee collected — no silently retained or silently lost dust.
- A fee-bearing trade and its mirror (buy-then-sell) can't be combined to pay less than two independently-applied fees would cost.
- An admin fee-rate change doesn't retroactively apply to trades already settled or escrowed.
- Fee accrual to "the beneficiary at trade time" uses the beneficiary as of execution, not one that can change mid-transaction via reentrancy.
- A referral/affiliate fee split (if present) can't be self-assigned by the trader to recapture their own fee.

### 2.10 Access control & authorization

- Every value-moving, parameter-changing, or lifecycle-transitioning function checks access control as its first operation.
- Every privileged role is scoped to exactly the powers it needs; no role transitively grants a broader one through a secondary path.
- A function gated to "the current owner" re-reads ownership on every call, never trusting a value captured earlier in the same transaction.
- Removing the last member of a required role set is blocked at the contract level, not just by convention.
- A role-gated check can't be bypassed via `delegatecall`, proxy storage-layout confusion, or a constructor/initializer that runs before access control is wired.
- Front-running an unprotected initializer can never grant the front-runner a privileged role the protocol later treats as legitimately configured.
- Every role-granting function is itself access-controlled to at least as strict a standard as the role it grants.

### 2.11 Reentrancy & call ordering

- State is fully updated before any external call that could reenter, for every function, not just the obviously risky ones.
- `nonReentrant` (or equivalent) guards every externally-reachable function that reads and writes shared mutable state across an external call boundary.
- Cross-function reentrancy (into a *different* function sharing state) is blocked, not just same-function reentrancy.
- Read-only reentrancy can't return stale/mid-transaction state to an external integrator in a way that mispriced something.
- A callback to a caller-supplied, unvalidated address happens only after all security-relevant state for the current call is finalized.
- Reentrancy guards use a correctly namespaced storage slot that survives an upgrade without collision (relevant for upgradeable proxies).

### 2.12 External price / oracle dependency

- Every external rate/price read is used consistently in one unit system throughout the call.
- A revert from the external oracle/rate source degrades gracefully (blocks only the dependent operation) rather than bricking unrelated state.
- An explicit non-zero, non-absurd sanity bound exists on any externally-read rate before it sizes a reserve, payout, or trigger.
- The external rate can't be moved within one transaction by the same caller who immediately benefits from having moved it.
- A thin, manipulable external source is read time-weighted, not as an instantaneous spot value.
- The external rate is read fresh on every use, or deliberately snapshotted once — never an unintentional mix within one logical operation.
- Staleness (last-updated timestamp vs. a heartbeat) is checked wherever the source provides one.

### 2.13 Token & asset integration

- Every external transfer checks its return value or uses a safe-transfer wrapper.
- Accounting is correct under a fee-on-transfer reserve asset, or such assets are explicitly and on-chain-enforced excluded.
- Accounting is correct under a rebasing reserve asset, or it's explicitly excluded.
- A token reverting on zero-value transfer doesn't brick a path that can legitimately compute a zero transfer.
- A transfer-hook token (ERC-777-style) can't reenter mid-trade in a way that violates any other invariant here.
- A pausable/blacklistable token causes, at worst, a DoS scoped to the affected address — never a cross-user fund loss.
- `decimals()` is read on-chain and used for scaling, never hardcoded, for every externally-supplied token.
- A token with multiple live addresses (legacy + current) doesn't confuse an allowlist or price source keyed by address.

### 2.14 Flash loans & same-block manipulation

- No state-transition trigger, or rate used for payout sizing, can be satisfied and reversed within a single transaction for a net profit.
- A flash-loan-funded trade is subject to identical fees, slippage checks, and caps as an organically-funded trade of the same size.
- Borrowing the reserve asset elsewhere, depositing, then withdrawing can't manufacture a claim on the protocol's own retained fees or reserves.
- Any on-chain-voting-weight-gated parameter is tested against a flash-loan-funded voting-weight spike.

### 2.15 MEV, sandwiching & front-running

- Every trade enforces a caller-specified minimum output (or maximum input) checked against the actual settled amount.
- No code path gives a privileged actor extra insertion power beyond what ordinary mempool visibility already gives any searcher.
- A batched/aggregated operation enforces per-item slippage protection, not just an aggregate check masking one bad fill among many good ones.
- A deadline parameter either exists on every trade function, or its absence is an explicit, documented decision.

### 2.16 Signatures & permits

- Every signature-gated function includes a nonce invalidated on use.
- Every signature includes (or is otherwise bound to) the chain ID.
- Every signature includes a deadline, checked against `block.timestamp` at execution.
- `ecrecover`'s return is checked against `address(0)` before being trusted.
- Signature verification rejects malleable (high-s) signatures.
- A failed permit (e.g. front-run) doesn't block the underlying operation when a sufficient standing allowance already exists.
- Every parameter that matters to the operation's outcome is included inside the signed message, not substitutable by the relayer.

### 2.17 Upgradeability & proxy safety

- Every upgrade is gated behind access control at least as strict as any other admin-critical action.
- Storage layout is append-only across upgrades, verified by a dedicated layout test or tool.
- Implementation contracts have their initializers disabled so they can never be initialized and operated directly.
- A proxy's initializer can be called exactly once, and a front-run of that call can't leave the proxy in a state the deployer's follow-up transactions silently fail to fully correct.
- An upgrade can't silently change the semantics of already-escrowed or already-committed state out from under users who committed under the old logic.

### 2.18 Admin, governance & centralization

- Every admin-only action has a severity-appropriate delay/multisig requirement, or is explicitly documented as a trusted, immediate-effect power.
- No admin action can directly seize user-owned funds (as opposed to protocol-owned fees/reserves) without consent.
- Transferring a high-value privileged role is a two-step (propose, accept) process.
- An admin parameter change can't retroactively disadvantage a position or order committed under the old parameters.
- Renouncing or burning an admin role is either disabled, or explicitly intended and confirmed not to brick a function that still requires it.

### 2.19 DoS & griefing resistance

- No function iterates an unbounded, user-controlled-length collection in a way gas can be driven above the block limit by a griefer.
- No single griefer-sized deposit can block, forever, a function that must remain callable for the protocol to function.
- A permissionless keeper function that reverts for one malformed target doesn't block processing of other targets.
- A malicious callback (revert-on-receive) can't indefinitely block a function the protocol needs to keep working for everyone else.
- Pausing one part of the system can't retroactively disadvantage users who acted before the pause.

### 2.20 Multi-actor, concurrency & cross-curve interactions

- Two independent instances sharing an underlying asset or dependency never leak state, escrow, or accounting between each other.
- An isolation snapshot correctly accounts for funds in flight from a third source at the exact moment of the snapshot.
- Concurrent operations on two different curves within the same block/transaction can't be sequenced to extract value neither operation alone would permit.
- A global admin action (rotating a shared dependency) doesn't silently invalidate in-flight state for every instance simultaneously in a way that strands funds.

### 2.21 Economic & game-theoretic design

- No participant can structure ordering, splitting, or timing to receive a better aggregate price than the formula intends for their true net trade size.
- A formula with more than one caller-influenceable input can't be manipulated by moving those inputs independently to create a favourable divergence.
- The protocol's stated incentive narrative actually holds under the literal contract logic, not just the intended story.
- No combination of otherwise-individually-safe operations composes into a profitable, loss-free cycle for any participant.

### 2.22 Gas & computational limits

- Every externally-reachable function has bounded, predictable gas that doesn't grow with unbounded user-controlled input past a chain's effective ceiling.
- A function split across phases specifically for a gas ceiling is safe to leave in the intermediate phase indefinitely, with no missable forcing window.

### 2.23 Chain-specific / infrastructure

- An L2 dependency on sequenced execution accounts for sequencer downtime/restart behaviour, where relevant.
- Block-time, finality, and reorg-depth assumptions match the actual target chain, not a carried-over mainnet default.
- Reliance on a specific EVM feature (e.g. transient storage) is confirmed live on every chain the contract deploys to.

### 2.24 Event & off-chain consistency

- Every state-changing function emits an event sufficient to reconstruct the resulting state without an indexer re-deriving it from ambiguous deltas.
- Event parameters match actual settled (post-fee, post-rounding) amounts, not pre-settlement requested amounts, wherever that's the reasonable off-chain expectation.
- A function with multiple logical sub-effects emits a distinct event per effect, not one conflated event.

### 2.25 Meta: verification invariants

- Every invariant in this document that applies to the protocol under audit has at least one corresponding executable test.
- Fuzz/property tests run at a realistic iteration count and input distribution, not a token smoke setting.
- A regression suite catches at least one deliberately-injected bug per critical invariant (mutation testing).
- Every "proven algebraically" claim has also been run as code at least once before being trusted in a report.

---

## 3. Attack vector catalogue

### 3.1 Sandwich & front-running
**Surface**: `buy()`/`sell()` with no caller-specified minimum output and/or no deadline.
**Mechanism**: attacker sees a pending trade, front-runs to shift price unfavourably,
back-runs to capture the difference.
**Mitigation**: require a caller-supplied `minOut` (never a hardcoded or optional-zero
default) checked against the *final* output, not an intermediate one; consider a
deadline parameter so a stale signed/submitted trade can't execute far later than
intended (informational on its own if `minOut` is enforced — it bounds the *price* risk,
just not the *timing*).
**Check**: does every value-moving entry point take and enforce a real `minOut`? Is it
checked after fees, not before?

### 3.2 Flash-loan curve manipulation
**Surface**: curves whose price depends on *live* `balanceOf()` reads (of the curve's
own reserve asset) rather than internally-tracked state.
**Mechanism**: flash-loan a large amount in, trigger a price-dependent action (mint,
graduate, borrow against the distorted price), reverse the loan in the same
transaction, keep the profit.
**Mitigation**: never derive price or triggering conditions from a live external
`balanceOf()` that a flash loan can move in the same block; derive from internally
updated state that only changes through the curve's own accounted trades. Where a
"real balance" read is unavoidable (e.g. capping an output at what's actually held),
make sure that read can only ever *reduce* an attacker's advantage, never create one.
**Check**: trace every price- or state-transition-relevant read back to its source — is
it curve-internal state, or an externally-movable balance? If a same-block round trip
(flash-loan funded or not) is mathematically possible, fuzz it directly (see §7) rather
than trusting a proof.

### 3.3 Whale-induced price collapse / pump-and-dump
**Surface**: curves with no trade-size caps or circuit breakers.
**Mechanism**: a large holder dumps, crashing price; either they (via an alt address)
or anyone else re-enters cheap while other participants are left holding a devalued
position.
**Mitigation**: this is largely a market-design question, not purely a contract bug —
every constant-product AMM has this property. It becomes a *contract* finding only if
the protocol's own documentation or design promises protection (caps, circuit
breakers, dynamic fees on steep price-impact trades) that the code doesn't actually
deliver, or if the "dump" is a round trip that nets the whale a profit (→ invariant 2).
**Check**: does the protocol claim any volatility protection? If so, does the code
enforce it? If not, confirm the round-trip-unprofitable invariant holds even at whale
scale (large single-trade sizes, not just many small ones).

### 3.4 Dust and zero-value inputs
**Surface**: `calculateBuyReturn()` / `calculateSellReturn()` and anywhere a formula
divides by a quantity that could be zero or near-zero.
**Mechanism**: a `0` or `1`-wei call breaks an implicit assumption, producing a
division-by-zero revert (DoS), an underflow, or — worse — a non-reverting path that
mints/pays out something for nothing.
**Mitigation**: explicit `amount > 0` checks at every entry point; precise,
overflow-safe math (`mulDiv`/`FullMath`/a vetted fixed-point library, never raw
`a * b / c` where `a * b` can overflow before the division saves it); test the exact
boundary (`0`, `1`, the largest value that doesn't yet trigger a cap) for every
formula, not just the entry points.
**Check**: for every formula, what's the smallest non-zero input, and what does the
formula do with it? Is there a code path where a non-zero state change happens for a
functionally-zero economic input?

### 3.5 Curve arithmetic inaccuracies
**Surface**: any curve using `pow`, `log`, `sqrt`, or a hand-rolled fixed-point
exponential/sigmoid, rather than plain integer constant-product math.
**Mechanism**: incorrect use of these operations (wrong precision, truncation in the
wrong direction, an untested extreme input) produces under/overflow or a rounding bug
that's much harder to reason about than linear math.
**Mitigation**: use audited fixed-point libraries (`PRBMath`, `ABDKMath64x64`, OZ's
`Math.sqrt`/`Math.mulDiv`), never a from-scratch implementation; unit-test every curve
math function across the full realistic input range plus the extremes, with an
explicit delta-tolerance check rather than exact equality.
**Check**: is every non-trivial math operation backed by a vetted library call? Is
there a test file dedicated to just the math, independent of the integration tests?

### 3.6 Price-oracle abuse (hybrid curves)
**Surface**: a curve whose price, sizing, or a state-transition trigger reads an
*external* price source (another protocol's token price, a TWAP, a Chainlink feed).
**Mechanism**: manipulate the external feed (thin-pool spot price, a short-TWAP
window, a token the attacker can mint/burn cheaply) to move the curve's price,
sizing, or triggering condition in the attacker's favour.
**Mitigation**: prefer a tamper-resistant source (TWAP over spot, a decentralized
oracle over a single pool read); where the external source is a protocol-native asset
(not a market price), establish *in whose units the curve accounts internally* — if
every quantity the curve tracks (reserves, the migration split, escrow) is denominated
in the external asset's own units rather than a derived USD value, then moving that
asset's *rate* can shift *when* something triggers, but can't let one participant
extract value from another, because nothing in the accounting ever converts back
through the manipulated number. Verify that claim explicitly rather than assuming it.
**Check**: enumerate every read of the external price/rate. For each one, is it used
only to *gate timing* (when does X happen), or does it also determine *how much* of
something a participant receives? The second case is where real loss lives.

### 3.7 Incomplete tax/fee enforcement
**Surface**: any `buy()`/`sell()` with a fee, tax, or spread, where more than one code
path can reach the underlying transfer.
**Mechanism**: a user finds a path that reaches the value-moving logic without going
through the fee-charging wrapper — a second unprotected entry point, an internal
function callable by a contract rather than the UI's intended one, interaction with a
non-standard (rebasing, fee-on-transfer) token that double-applies or skips the fee.
**Mitigation**: fee logic lives in exactly one place that every path provably funnels
through (verify this with a grep for every caller of the underlying transfer/trade
function, not just a read of the "normal" path); gate any lower-level, fee-free entry
point behind strict access control reachable only by trusted, protocol-owned callers;
confirm the traded assets are either protocol-controlled (no non-standard-token risk)
or explicitly vetted.
**Check**: is there more than one external/public function that can move the curve's
core asset? If so, does every one of them apply the fee, or is at least one
access-gated to exclude ordinary users?

### 3.8 Novel curve exploits ("bad theory, good implementation")
**Surface**: any formula built on an economic/game-theoretic assumption rather than
pure conservation math — e.g. a price derived as one side's total divided by a
*shared* total both sides can independently influence.
**Mechanism**: the code does exactly what it says, but what it says can be gamed. The
canonical example (Ethos Network): `price(side) = votes[side] * basePrice /
(votes[TRUST] + votes[DISTRUST])`. An attacker alternates bets on *both* sides to
suppress the ratio from moving, then unwinds one side to realize an asymmetric profit
the "balanced" price never signalled.
**Mitigation**: there's no checklist item that catches this — it requires treating the
formula as an adversary would: can a participant influence the *denominator* and the
*numerator* independently, in a way that lets them choose a favourable moment to
diverge? Simulate the game, don't just audit the code. For curves with a single
asset pair (no shared denominator between competing outcomes), this specific shape
doesn't transfer, but the *method* — "what can a participant do to the formula's
inputs, not just its enforcement?" — applies to any economic design.
**Check**: write down every variable the formula reads. For each one, who can move it,
and does moving it in combination with timing create a state the formula's author
didn't model? This is the one category worth spending unstructured, adversarial
thinking time on rather than working a checklist.

### 3.9 MEV & ordering attacks (beyond basic sandwich)

- **JIT liquidity insertion**: deposit immediately before a large trade, withdraw immediately after, capturing fee/price-impact exposure without holding risk.
- **Graduation back-running**: be the first trader on a newly opened destination venue at the instant it opens.
- **Multi-block MEV**: split an attack across several blocks to evade guards that only check intra-block conditions.
- **Priority-gas racing** for a profitable permissionless state transition (e.g. a keeper reward for finalizing).
- **Reorg-exploiting replay** on shallow-finality chains: let a profitable transaction get reorg'd out, then replay it once favourable.

### 3.10 Flash-loan & capital-efficiency attacks

- **Flash-mint** (not just flash-loan) of the reserve or an LP-equivalent token, if it supports flash-minting, for single-block distortion with no external lender needed.
- **Flash-loan-funded governance attack**: borrow enough voting weight to pass a malicious parameter change and execute it within one transaction, if any admin parameter is gated by on-chain voting weight.
- **Recursive cross-protocol flash-loan chaining** to amplify single-block capital beyond what any one lender would extend alone.

### 3.11 Oracle & external-dependency attacks

- **Thin-pool spot manipulation**: move a low-liquidity pool's price cheaply relative to the value it unlocks.
- **TWAP window gaming**: sustain a distorted price just long enough to bias a short averaging window.
- **Oracle-downtime DoS**: exploit an asymmetric failure mode on oracle revert to target a specific victim.
- **Depeg exploitation**: treat a wrapped/derivative asset as always worth its peg, redeeming at the stale rate during a depeg.
- **Decimals mismatch**: two feeds at different decimal precision silently treated as the same scale.
- **Source governance takeover**: compromise the price source itself, if it's protocol-controlled rather than a decentralized feed.

### 3.12 Reentrancy variants

- Classic single-function reentrancy via an external call before a state update.
- Cross-function reentrancy exploiting state shared between two entry points.
- Cross-contract reentrancy via a second contract in the same system trusting the first's mid-call state.
- Read-only reentrancy: an external integrator reads a view function mid-reentrancy and gets stale data.
- ERC-721/1155 receive-hook reentrancy during a transfer nested inside a larger state-changing operation.
- Permit-callback reentrancy: a malicious token's `permit()`/`transferFrom()` hook reenters before the caller's own checks complete.

### 3.13 Access control & privilege attacks

- Unprotected/front-runnable initializer on a non-proxy or improperly guarded proxy.
- A missing `onlyOwner`/role check on a function clearly intended to be gated (compare against gated siblings).
- Tautological access check (checking a parameter the attacker controls instead of a fixed, pre-set role).
- Privilege escalation via a secondary, less-obviously-sensitive function that can grant the primary privileged role.
- `delegatecall` to an attacker-influenceable address, executing arbitrary logic in the caller's storage context.
- Storage-layout confusion across a proxy upgrade causing an access-control flag to read from the wrong slot.

### 3.14 Token & asset-specific attacks

- Fee-on-transfer token causing the protocol to credit more than it actually received, creating a deficit later users inherit.
- Rebasing token silently inflating/deflating the held balance relative to the internally tracked reserve.
- A malicious/overridden `decimals()` returning a different value than the integration assumed at deploy time.
- A multiple-address token (legacy + current both live) confusing an allowlist or address-keyed price source.
- ERC-777 `tokensReceived` used as a reentrancy vector inside what looks like a simple transfer.
- A token returning `false` instead of reverting on a failed transfer, silently accepted by a non-safe-transfer call site.

### 3.15 Precision & rounding exploitation

- Donate-then-redeem share inflation: first-depositor donates directly (not via `deposit()`), rounding the second depositor's shares to zero.
- Compounding a per-transaction protocol-favourable rounding across many transactions in a way that flips net-favourable due to a missed asymmetry.
- Forcing a formula into a branch where integer division truncates a fee to exactly zero, below the intended minimum.
- Exploiting a silently-wrapping downcast instead of a checked/`SafeCast`-style one.

### 3.16 Lifecycle & state-machine attacks

- Forcing a premature transition by satisfying only the letter, not the economic intent, of the cheapest available trigger.
- Re-triggering a one-time transition via a missing idempotency check, double-applying its side effects.
- Exploiting the gap between two phases of a split transition to insert an action one phase assumed the other had already foreclosed.
- Stalling a transition indefinitely by keeping a precondition just barely unmet, when staying pre-transition is more profitable.

### 3.17 Migration / venue-transition attacks

- Pre-seeding the destination venue with a skewed ratio before migration, to claim a disproportionate share of what gets deposited.
- Front-running the permissionless finalize call to be the first trader on the newly opened venue.
- Donating dust specifically to manipulate which code branch the migration logic takes (empty-pool vs. already-liquid).
- Creating the destination pool directly via the underlying DEX factory with attacker-chosen initial parameters, if the protocol doesn't verify pool state before trusting it.

### 3.18 Fee & tax evasion

- Routing through an unprotected or differently-gated entry point that reaches the same transfer without the fee wrapper.
- Structuring a trade to land exactly on a rounding boundary that zeros the fee.
- A flash-loan-funded round trip extracting value equal to or exceeding the fee nominally charged, netting a profit.
- Exploiting a fee exemption meant for one privileged caller by impersonating or proxying through it.

### 3.19 Governance & admin-trust attacks

- A compromised admin using an instant (non-timelocked) parameter change to front-run a disadvantageous trade against users.
- A malicious admin setting an extreme parameter for one block, extracting value, then reverting it before anyone notices.
- Key-compromise of a single-signer role where documentation implies multisig protection that isn't actually configured on-chain.

### 3.20 DoS & griefing

- Unbounded-loop griefing: inflate an array/set the protocol must iterate until gas exceeds the block limit.
- Griefing a permissionless keeper/finalize function by making it revert for one target, if failures aren't isolated per-target.
- Donation-based griefing: send a trivial amount to flip a zero-balance check used as a state-machine signal.
- Malicious-receiver griefing: a contract that reverts on receipt, specifically to block a function that must pay it to proceed.
- Front-running a legitimate user with a dust transaction that changes which branch they land in, at extra cost, with no direct extraction (pure griefing).

### 3.21 Signature & authentication attacks

- Replaying a valid signature after the state it was predicated on changed, because no nonce was bound to that state.
- Cross-chain replay on a second deployment of the same contract, because chain ID wasn't included.
- Signature malleability: a second valid signature derived for the same message without the private key.
- Omitting a critical parameter from the signed message, letting a relayer substitute a different value.
- Phishing a signer whose EIP-712 human-readable summary doesn't match its actual on-chain effect.

### 3.22 Upgrade & proxy attacks

- Storage collision introduced by an upgrade that doesn't preserve the prior layout.
- Upgrading to an implementation whose initializer can be re-run, re-granting admin rights to an attacker.
- Function-selector clashing between a proxy's own admin functions and the implementation's, routing a crafted call to the wrong one.
- An uninitialized implementation contract called directly (no proxy) and hijacked, if it's still the address a working proxy delegates to.

### 3.23 Multi-curve / cross-protocol composability attacks

- Shared-dependency drain: a bug in one instance reaching into funds actually escrowed for a sibling instance sharing the same underlying asset.
- Cross-protocol price manipulation: moving this protocol's price specifically because a *different*, composable protocol reads it as an input.
- Using this curve as a cheap, manipulable leg in a larger cross-protocol arbitrage or liquidation attack aimed at a different system.

### 3.24 Economic & game-theoretic exploits

- Alternating-side manipulation of a shared-denominator pricing formula (the Ethos-style exploit) to suppress a signal, then unwinding for an asymmetric profit.
- Reflexive-design gaming: simulating a rewarded behaviour (e.g. "long-term holding") only long enough to capture the reward, then reversing it.
- Coordinated multi-account Sybil attacks against any mechanism assuming one-address-one-actor (a per-address cap, a first-depositor bonus).
- Profiting from the *opposite* of a mechanism's intended action whenever its literal logic accidentally rewards that instead.

---

## 4. Audit checklist

Not exhaustive — bonding curves vary widely in design. Always read the specific
formula in its own business context before treating any of these as satisfied. §4.1 is
the essential short list; §4.2 onward is an extended, categorized checklist mirroring
§2 and §3 — for working a design exhaustively.

### 4.1 The essentials

- [ ] **Precision**: every formula handles decimals exactly; no mixed-precision math
      (6-decimal and 18-decimal tokens combined without explicit scaling)
- [ ] **Multiplication before division**, always, with overflow-safe intermediate math
- [ ] **Zero-value inputs** either revert explicitly or are proven no-ops
- [ ] **Extreme inputs** (smallest non-zero, largest realistic, and the theoretical max)
      tested against every formula, not just the entry points
- [ ] **Token compatibility**: confirm which assets the curve trades are
      protocol-controlled vs. externally arbitrary; for arbitrary ones, check
      fee-on-transfer, rebasing, pausable, and deny-list (blacklist) behaviour
- [ ] **Initialization state**: can the curve be used, or can its constant be set, in a
      way that bricks it or lets the deployer front-run their own setup?
- [ ] **Rounding direction**: every rounding favours the protocol/other participants,
      never the caller; verify this holds on *both* sides of every paired operation
      (buy vs. sell, mint vs. burn) — see "symmetry" audits for paired-operation bugs
- [ ] **Stuck funds**: enumerate every address that can end up holding curve assets
      (the curve itself, an escrow, a migration destination) and confirm each has a
      retrieval path under every reachable state, including adversarial pre-seeding
- [ ] **DoS resilience**: can any caller force a revert in a path that *must* succeed
      for the protocol to function (e.g. a permissionless migration/finalization step)?
- [ ] **Volatility / round-trip**: fuzz many round trips at varied, realistic sizes —
      not one fixed amount — and assert the trader never nets a profit (see §7)
- [ ] **Lifecycle transitions**: for every state-dependent check, confirm it reads live
      state; confirm the transition itself can't be triggered, delayed, or skipped by
      an unprivileged caller in a way that changes who benefits
- [ ] **Access control parity**: for every privileged/unprivileged pair of paths to the
      same effect (e.g. an allowlisted direct caller vs. the normal wrapper), confirm
      fees, caps, and checks match on both sides

### 4.2 Curve math & pricing

- [ ] Marginal price verified strictly positive and finite across the full input range, including at the supply ceiling
- [ ] Output monotonicity tested with increasing-input pairs at the same state
- [ ] Inverse formula (if any) cross-checked against the forward formula for consistency
- [ ] No input produces an output exceeding the theoretical maximum
- [ ] Trade-splitting tested and shown not to beat the single-trade price
- [ ] Virtual and real reserves confirmed never confusable in any payout-computing path

### 4.3 Conservation & balances

- [ ] `sum(user claims) == internal total-supply counter` asserted as a standing invariant test
- [ ] `assets in == assets out + fees + still-escrowed` checked after every state-changing call
- [ ] Direct donation tested and shown not to inflate any individual user's claim
- [ ] Double-claim tested and confirmed to revert/no-op on the second attempt
- [ ] Share-to-asset round trips tested and shown never to let a user extract more than they put in

### 4.4 Reserves vs. real balance

- [ ] `balanceOf(curve) >= storedReserve - virtualOffset` fuzzed across arbitrary donation amounts
- [ ] `sync()`-equivalent (if present) tested against every pre-seeding shape
- [ ] Dust donations tested against every conditional branch that reads a real-balance check

### 4.5 Boundary & dust values

- [ ] `buy(0)`/`sell(0)` tested explicitly for revert or state-no-op
- [ ] 1-wei trades tested explicitly
- [ ] `type(uint256).max` (and the asset's actual max balance) tested against every amount parameter
- [ ] Every threshold's exact boundary tested: landing on it, one unit below, one unit above
- [ ] First-ever trade on a fresh curve tested against the formula's prediction
- [ ] Last reachable unit of supply tested for correct, non-stranding behaviour

### 4.6 Rounding & precision

- [ ] Every division confirmed preceded by its multiplication
- [ ] Rounding direction verified favourable-to-protocol on both sides of every paired operation
- [ ] Fee computation tested at the smallest size that should still produce a non-zero fee
- [ ] Decimal scaling tested with real mismatched-decimal tokens, confirmed applied exactly once
- [ ] `mulDiv`-style wide math confirmed used wherever a multiply precedes a divide
- [ ] Downcasts tested at the exact overflow boundary, confirmed checked/safe

### 4.7 Lifecycle & state machine

- [ ] Every documented transition has a "cannot go backward" test
- [ ] Every state-gated function tested for clean revert (not partial execution) in the wrong state
- [ ] Lifecycle view functions tested for mutual exclusivity
- [ ] Transition triggers confirmed to read live state via a test that changes state mid-flow
- [ ] `nonReentrant` confirmed present on every lifecycle-transitioning function

### 4.8 Migration / graduation

- [ ] Zero-gap-at-open tested against every pre-seeding shape (empty, dust-synced, meaningful, swap-forcing)
- [ ] Pre-seeder profit tested and shown bounded at or below their own deposit
- [ ] Permissionless finalize step tested for front-run resistance and safe-indefinite-delay
- [ ] Cross-instance escrow isolation tested explicitly with two instances sharing an asset
- [ ] Post-migration dust disposal tested; nothing strandable confirmed

### 4.9 Fees & tax

- [ ] Every value-moving entry point enumerated and confirmed fee-applying or intentionally fee-exempt-and-gated
- [ ] Fee-rate ceiling enforced on-chain, tested at and above the maximum
- [ ] Fee split arithmetic tested to sum exactly, including at dust sizes
- [ ] Buy-then-sell round trip tested to confirm it can't net a profit from the fee itself
- [ ] Fee-rate admin changes tested to confirm no retroactive repricing

### 4.10 Access control

- [ ] Every sensitive function's access check confirmed first-operation by direct code read
- [ ] Every role tested for reaching a broader role via a secondary path
- [ ] "At least one must remain" constraints tested at the boundary
- [ ] Initializers tested for front-run resistance and the consequence if front-run
- [ ] Proxy storage layout verified with a dedicated diff test across every upgrade

### 4.11 Reentrancy

- [ ] Checks-effects-interactions order verified by code read for every function with an external call
- [ ] `nonReentrant` coverage verified complete, including "read-mostly" functions that write shared state
- [ ] Cross-function reentrancy tested with a malicious token/contract reentering a different function
- [ ] Read-only reentrancy considered for every view function an integrator might call mid-transaction
- [ ] Callbacks to caller-supplied addresses confirmed to run after all security-relevant state is finalized

### 4.12 Oracle & external dependencies

- [ ] Every external rate read enumerated, unit system documented and verified consistent
- [ ] Oracle revert paths tested for graceful degradation vs. brick
- [ ] Sanity bounds tested on every externally-read rate before use
- [ ] Same-block rate manipulation tested against every function reading the rate
- [ ] TWAP/time-weighted reads confirmed used wherever the source is thin or manipulable

### 4.13 Token & asset integration

- [ ] Every external transfer checked for return value or wrapped safely
- [ ] Fee-on-transfer and rebasing scenarios tested or verifiably excluded
- [ ] Zero-transfer-reverting token tested against every code path that can compute a zero transfer
- [ ] Transfer-hook tokens tested for hook-based reentrancy
- [ ] Pausable/blacklistable token tested to confirm scoped blast radius
- [ ] `decimals()` confirmed read on-chain, never hardcoded, for every external token

### 4.14 Flash loans

- [ ] Every state-transition trigger tested against a same-transaction borrow-and-repay sequence
- [ ] Fees/caps/slippage confirmed identical for flash-loan-funded vs. organic trades
- [ ] Voting-weight-gated parameters tested against a flash-loan-funded spike

### 4.15 MEV & front-running

- [ ] Every trade function confirmed to enforce a real caller-specified min-out/max-in
- [ ] Batched/multicall operations tested for per-item, not just aggregate, slippage protection
- [ ] Deadline parameter present, or its absence an explicit recorded decision

### 4.16 Signatures & permits

- [ ] Nonce-based replay protection tested on every signature-gated function
- [ ] Chain-ID inclusion tested, or cross-chain replay shown impossible another way
- [ ] Deadline enforcement tested, including the exact-boundary case
- [ ] `ecrecover` return checked against `address(0)` in every raw usage
- [ ] Malleability resistance confirmed, tested with a known malleable signature pair
- [ ] Permit-fails-but-allowance-sufficient path tested to confirm the operation still succeeds

### 4.17 Upgradeability

- [ ] Storage layout diffed across every upgrade in CI or as a standing test
- [ ] Implementation contracts confirmed to have initializers disabled
- [ ] Upgrade access control tested identically to every other admin-critical action
- [ ] In-flight escrowed/committed state tested against silent semantic change on upgrade

### 4.18 Admin & governance

- [ ] Every admin action's delay/multisig requirement (or explicit absence) verified on-chain, not just in prose
- [ ] Admin actions confirmed unable to directly seize user-owned funds
- [ ] High-value role transfers confirmed two-step
- [ ] Parameter changes tested for no retroactive effect on committed positions
- [ ] Role renunciation/burning tested for effect on required future operations

### 4.19 DoS & griefing

- [ ] Every loop over a user-controlled-length collection bounded or paginated, gas-tested at large size
- [ ] Dust-sized griefing deposits tested against every zero-balance/threshold state signal
- [ ] Permissionless keeper functions tested for per-target failure isolation
- [ ] Malicious-receiver contracts tested against every must-pay-out-to-proceed function

### 4.20 Multi-actor / cross-curve

- [ ] Shared-dependency isolation tested explicitly between two instances under interleaved operations
- [ ] Global admin actions tested for effect on every live instance simultaneously
- [ ] External integrators' reliance on this protocol's state as a price/trust signal considered and its manipulability checked

### 4.21 Economic design

- [ ] Every caller-influenceable formula input enumerated, checked for independent-influence manipulation
- [ ] Stated incentive narrative checked directly against literal contract logic
- [ ] Sybil resistance considered for every one-address-one-actor assumption

### 4.22 Gas & limits

- [ ] Gas measured for every externally-reachable function at a realistic worst-case input size
- [ ] Multi-phase, gas-ceiling-driven functions tested for safety of indefinite delay in the intermediate phase

### 4.23 Chain-specific

- [ ] L2 sequencer behaviour considered and tested or documented out of scope, if applicable
- [ ] Block-time/finality/reorg assumptions confirmed to match the actual target chain
- [ ] Chain-specific EVM features confirmed live on every deployment target

### 4.24 Events & off-chain consistency

- [ ] Every state-changing function's event checked against indexer reconstruction needs
- [ ] Event amounts confirmed to reflect settled, not requested, values
- [ ] Multi-effect functions checked for distinct events per effect

### 4.25 Testing & process (meta)

- [ ] Every applicable invariant from §2 mapped to at least one executable test
- [ ] Fuzz/property tests run at realistic iteration count and input distribution
- [ ] Mutation testing performed: at least one injected bug per critical invariant, confirmed caught
- [ ] Known-issue/prior-audit register cross-checked against every candidate finding before reporting

### 4.26 Documentation & disclosure

- [ ] Every NatSpec/comment-claimed invariant verified true by a test, not just asserted in prose
- [ ] Every documented trust assumption made explicit and checked against what the code actually enforces vs. merely assumes
- [ ] Known trade-offs and accepted risks distinguished clearly from unverified assumptions in the audit's own output

### 4.27 Deployment & operational

- [ ] Deployment script reviewed with the same rigor as the contracts — constructor args, init order, role bootstrap sequence
- [ ] Multi-transaction deployment sequences checked for front-runnable gaps between transactions
- [ ] Post-deployment configuration (role grants, parameter sets) verified to match what the audited code assumes before value is allowed to flow

---

## 5. Precision & token-integration notes specific to curves

- **Decimal mismatches compound across a curve's lifecycle.** A curve seeded in one
  asset's decimals and migrated into a venue denominated differently needs every
  conversion point audited independently — a single missed `/1e12`-style scale factor
  anywhere in that chain silently breaks pricing, not just reverts.
- **Rounding direction must be independently verified for every one of a curve's
  formulas** (buy, sell, the migration split, fee proration) — a formula can be
  internally consistent and still leak value if one rounds toward the protocol and
  the "inverse" formula also rounds toward the protocol instead of mirroring it.
- **Non-standard tokens are a curve-specific risk multiplier**: a fee-on-transfer or
  rebasing token held as curve reserve breaks invariant 3 (`stored == real`) the
  moment it's donated to or rebases underneath the curve, independent of any bug in
  the curve's own math.

---

## 6. Oracle-hybrid curves

A curve that prices or sizes itself against an *external* asset's rate (rather than
being purely self-contained) inherits that asset's oracle-integration risks on top of
its own curve risks. Treat the external rate exactly like any other price oracle:

- Is it checked for staleness, or read live on every call? (Live reads avoid
  staleness but inherit whatever manipulation window the external source has.)
- Is there a sanity bound (non-zero, not absurdly large) before it's used to size a
  launch or a reserve?
- What happens on a revert from the external call — does it degrade gracefully
  (blocking only that asset's curves) or does it brick something shared?
- Critically: **does the rate gate *timing* only, or does it also determine *amount*
  paid to one participant at another's expense?** A rate used only to decide *when*
  a threshold is crossed is far lower-risk than one used to compute a payout, because
  manipulating a timing-only gate can shift *when* something happens but can't, on its
  own, transfer value between participants if everything downstream stays denominated
  in the external asset's own units rather than a value the manipulated rate converts
  through.

---

## 7. Fuzzing & PoC methodology

- **Don't trust an algebraic proof that a property holds — run it.** "The formula
  always recomputes from the fixed constant, so rounding can't accumulate" is a claim;
  hundreds of real round trips at varied, realistic sizes (not one fixed amount
  repeated) either confirm it empirically or falsify it. A proof that's never been run
  as code is a hypothesis.
- **Vary the input, not just the iteration count.** "High volume" (many iterations) and
  "realistic" (varied, non-constant trade sizes) are independent axes — a thousand
  rounds of the exact same size explores far less of the state space than a hundred
  rounds of genuinely different sizes.
- **Prove boundaries by construction, not search.** Where a specific gap is suspected
  (e.g. two formulas that should agree but read state at different times), compute the
  exact input that lands in the gap directly from the two formulas' definitions —
  binary-search against a pure/view preview function if the forward formula isn't
  directly invertible — rather than hoping a random fuzzer stumbles onto it.
- **On every reverted adversarial attempt, assert nothing moved.** A finding isn't
  closed by "it reverts" alone — confirm balances and reserves are byte-identical
  before and after the revert, so a partial-state-change bug can't hide behind an
  expected-revert test that never checked for side effects.
- **Mutation-test the test suite itself.** Inject a single-line regression into a
  scratch copy of the contract (flip a rounding direction, remove one guard) and
  confirm the suite actually fails. A passing suite that can't catch an obvious
  injected bug isn't evidence of anything.

---

