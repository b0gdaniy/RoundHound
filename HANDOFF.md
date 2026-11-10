# RoundHound — Olympix take-home build brief (handoff to local Claude Code)

This is the full spec for the proof-of-concept. Hand this file to Claude Code as the project
brief, then build the repo locally where `forge`, `slither`, and `solc` are available.
The sandbox that drafted this was network-isolated, so nothing here has been compiled yet —
but the **core math is already verified** in `sim/stableswap_sim.py` (run it first).

---

## 0. The pitch (one paragraph)

On Nov 3 2025, Balancer v2 lost ~$128M to a rounding-direction bug that had survived multiple
audits since 2021. Every individual swap was correct; the bug only became exploitable as the
_cumulative_ effect of dozens of swaps against a composable pool deflated to wei-scale balances.
This is exactly the class of bug that unit tests and single-operation audits miss. **RoundHound**
is a developer-time tool (shift-left, CI-friendly — Olympix's thesis) that catches it three ways:
static rounding-direction triage, invariant-driven boundary-aware fuzzing, and a mutation oracle
that proves the test suite is actually adequate.

---

## 1. Verified root cause (use this exact framing in the writeup)

Vulnerable path: `ComposableStablePool` EXACT_OUT swap (`_swapGivenOut`).

```solidity
_upscaleArray(balances, scalingFactors);
swapRequest.amount = _upscale(swapRequest.amount, scalingFactors[indexOut]); // BUG
uint256 amountIn = _onSwapGivenOut(swapRequest, balances, indexIn, indexOut);
amountIn = _downscaleUp(amountIn, scalingFactors[indexIn]);                   // correct: rounds up
return _addSwapFeeAmount(amountIn);
```

`_upscale` uses `FixedPoint.mulDown` (rounds DOWN) on the OUTPUT amount. It should round UP.
Rule: rounding must always favor the protocol. Rounding the user's output down understates how
much they take, so the pool computes a smaller required `amountIn`, and the invariant `D` ends up
lower than it should. Off-by-one at normal liquidity. Composability is the amplifier: the pool
holds its own BPT as a swappable asset, and the attacker can enter a temporary BPT _deficit_
(no upfront BPT needed), trade BPT for the pool's tokens to deflate liquidity to wei-scale, where
the rounding error becomes a large percentage, then settle the cheap deficit and pocket the
difference. Flash-loaned capital, constructor-based, ~65 iterations in one atomic batchSwap.

Sources: Certora (identified root cause same day), Trail of Bits (audited Balancer since 2021;
flagged this exact class as TOB-BALANCER-004 in 2021, "undetermined severity"; Stable Math was
later out of scope). Both unambiguously attribute it to rounding direction — NOT access control.

### On-chain forensics (real, pulled from mainnet — strong writeup material)

- Stage 1 (theft): tx `0x6ed07db1…bc9742`, block 23717397, 2025-11-03 07:46:47 UTC.
  `to: null` (contract creation -> constructor attack), gas used **28,348,674** (~full block =
  the ×65 loop), `token_transfers_overflow: true`, decaying BPT-fee mints to the Protocol Fees
  Collector `0xce8868…9f9F` (1.38, 0.0138, 0.00235, 0.000138 … = the iterative-rounding
  fingerprint), `has_error_in_internal_transactions: true` (boundary probing).
- Stage 2 (extraction): tx `0xd155207…b48569`, 84 seconds later, method `0x8a4f75d6`.
  Vault `0xBA12…2C8` -> recipient `0xAa76…8e3f`: 6587.44 WETH, 6851.12 osETH, 4259.84 wstETH,
  plus dust BPT (44.15 osETH/wETH-BPT, **20.41 wstETH-WETH-BPT**). Exited with thousands of real
  tokens and ~zero BPT — the signature of a settled deficit round-trip.
- Actors: deployer/EOA `0x506D…3207`, exploit contract `0x54B5…a30d`, recipient `0xAa76…8e3f`.
- Pools hit: osETH/wETH-BPT `0xDACf5Fa19b1f720111609043ac67A9818262850c`,
  wstETH-WETH-BPT `0x93d199263632a4EF4Bb438F1feB99e57b4b5f0BD`.

---

## 2. Why existing tools missed it (this is the novelty argument)

- `roundme` (Trail of Bits) does rounding-direction analysis but is **human-assisted** — a person
  must declare the intended direction per operation. RoundHound _derives_ the required direction
  from a declared invariant.
- Certora's 2022 v2 proofs were **solvency-only** (sum of BPT <= totalSupply); they did not
  constrain _per-swap rounding_ or _roundtrip_ behavior, so the bug slipped through.
- Stock fuzzers (Echidna/Medusa/Foundry) almost never drive a balance to single-digit wei out of
  the 2^256 space, so a naive invariant suite passes. RoundHound **biases the corpus** to the
  boundary where the bug lives.

---

## 3. Architecture (three layers)

**Layer 1 — static triage (Slither, Python).** Detector that finds scaling/invariant code paths
(`_upscale`, `_downscale`, `mulDown`, `mulUp`, `divUp`, `divDown`) and flags any output-side /
invariant-feeding amount rounded _down_. Framed honestly as heuristic triage that points the
fuzzer, not a sound verifier. Improvement over `roundme`: expected direction inferred from a
declared `@invariant` NatSpec tag rather than from a human.

**Layer 2 — invariant-driven boundary-aware fuzzing (the core; the part that actually catches it).**
Foundry invariant tests generated from a tiny spec:

- `prop_noFreeMoney`: a token roundtrip (out then back) never increases the actor's net balance
  (Certora-validated "roundtrip swap invariance").
- `prop_invariantNonDecrease`: `D` never decreases across a swap beyond a fixed wei tolerance.
- `prop_shareValueMonotone`: per-BPT share value never decreases on any user op.
  Plus a **boundary-biased handler**: the fuzz handler seeds balances/amounts from a dictionary of
  wei-scale values, scaling-factor edges, and deficit states. Demo: stock handler PASSES (misses),
  boundary handler FAILS (finds + prints the exploit sequence).

**Layer 3 — mutation oracle (Olympix flourish).** Flip rounding directions
(`mulDown`<->`mulUp`, `divUp`<->`divDown`) via `slither-mutate` (or the bundled script), run the
invariant suite, report surviving mutants = inadequate rounding tests. Show the boundary suite
kills the mutant while a naive unit test does not.

---

## 4. Repo layout to build

```
roundhound/
├── README.md                       # the writeup (English) — see section 6
├── sim/stableswap_sim.py           # DONE, verified — keep, reference in README
├── foundry.toml
├── src/
│   ├── lib/FixedPoint.sol          # mulDown/mulUp/divUp/divDown (port minimal)
│   ├── lib/StableMath.sol          # get_D / get_y (port from sim)
│   └── VulnerableStablePool.sol    # _upscale uses mulDown (BUG); fix flag via constructor or 2 files
├── test/
│   ├── handlers/StockHandler.sol   # uniform random swaps (misses the bug)
│   ├── handlers/BoundaryHandler.sol# wei-scale / deficit-biased swaps (finds it)
│   ├── Invariants.t.sol            # the 3 properties, run against both handlers
│   └── Exploit.t.sol               # deterministic PoC replay (prints attacker profit)
├── detector/roundhound_detector.py # Layer 1 Slither custom detector
└── mutation/run_mutations.sh       # Layer 3 driver (slither-mutate or sed-based fallback)
```

Tip: implement the fix as a second contract `FixedStablePool.sol` (or a `bool buggy` immutable)
so the same test file demonstrates before (red) and after (green) without editing tests.

---

## 5. Build & run commands (local)

```bash
# verify the math first (no deps)
python3 sim/stableswap_sim.py

# scaffold
forge init --no-git . 2>/dev/null || true
forge build

# Layer 2 — the headline demo
forge test --match-contract Invariants -vvv          # boundary handler: FAIL on vulnerable
forge test --match-test test_exploit_replay -vvv     # prints attacker profit

# after applying the fix (FixedStablePool / buggy=false)
forge test -vvv                                       # all green

# Layer 1
pip install slither-analyzer --break-system-packages
slither . --detect roundhound-rounding-direction      # register custom detector

# Layer 3
bash mutation/run_mutations.sh                         # shows surviving vs killed mutants
```

## 5a. Acceptance criteria (definition of done)

1. `sim/stableswap_sim.py` prints leak=0 (correct) vs leak>0 (buggy) at low liquidity. [DONE]
2. Boundary handler makes `prop_noFreeMoney` / `prop_invariantNonDecrease` FAIL on the vulnerable
   pool; stock handler does NOT (proves boundary biasing matters).
3. `test_exploit_replay` prints a positive attacker profit on the vulnerable pool, ~0 on the fixed.
4. Fixed pool: full suite green.
5. Mutation run: at least the `mulUp->mulDown` mutant is KILLED by the boundary suite; show a naive
   unit test that the same mutant SURVIVES (the point of Layer 3).
6. `forge build && forge test` clean before declaring done.

---

## 6. Writeup (README.md) outline — English, submission-facing

1. Exploit chosen + why (rounding bug that beat 4 years of audits; the testing-philosophy gap).
2. Root cause, code-level (section 1 above) + the BPT price dip/recover timeline.
3. On-chain forensics with the two real tx hashes (section 1 forensics).
4. Why existing tooling missed it (section 2).
5. RoundHound architecture (section 3) + how each layer maps to a specific miss.
6. PoC results: sim output, stock-vs-boundary fuzz, exploit replay numbers, fix, mutation kills.
7. Limitations & honesty: static layer is heuristic triage; fuzzing needs the invariant spec;
   future work (auto-derive invariants, formal-verification handoff).

Keep prose clean (no AI tells) — this goes to Mason at Olympix.

---

## 7. Paste-this prompt for local Claude Code

> Build the RoundHound repo per HANDOFF.md in this folder. Start by porting sim/stableswap_sim.py
> to src/lib/StableMath.sol and src/lib/FixedPoint.sol, then VulnerableStablePool.sol with the
> `_upscale` mulDown bug. Write the Foundry invariant suite (3 properties) plus StockHandler and
> BoundaryHandler, and a deterministic Exploit.t.sol that prints attacker profit. Run
> `forge build && forge test` and iterate until the acceptance criteria in section 5a pass. Then
> add the Slither detector (Layer 1) and mutation driver (Layer 3). Match the verified root cause
> framing exactly; do not invent metrics.

```

```
