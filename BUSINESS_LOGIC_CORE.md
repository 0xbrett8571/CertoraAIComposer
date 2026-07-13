# Business Logic Core — Updated Sections for MASTER_AUDIT_GUIDE_V2.md
## Replaces Parts 3, 4, 6 (Prompt A), and 13

> These sections replace the corresponding parts in MASTER_AUDIT_GUIDE_V2.md.
> Foundation: arXiv:2507.20175 — analysis of $1B+ real-world DeFi losses.
> Classical bugs (reentrancy, overflow) are table stakes. Business logic is the differentiator.
>
> **Grounded in AIComposer's actual system prompts**: The property categories
> and extraction patterns below align with `property_analysis_system_prompt.j2`
> and `property_analysis_prompt.j2` — the prompts AIComposer uses internally
> for auto-prove Phase 4. The Prompt A in Part 6 matches the extraction
> patterns the system uses. The Auditor Mindset filter maps directly to
> the "reasoning field" requirement in the system prompt: *"Your reasoning
> field is load-bearing. Be specific about what you looked at."*
>
> **See also**: `composer/templates/cvl_guidelines.j2` for the 23 CVL rules
> the CVL Judge enforces. `examples/cccp_buggy/` for a gold-standard
> design document + CVL spec.

---

## PART 3 (UPDATED): THE REAL THREAT LANDSCAPE

### Why Classical Bugs Are No Longer Enough

Modern automated tools (Slither, MythX, Echidna) catch most classical bugs:
- Reentrancy
- Integer overflow/underflow
- Unchecked return values
- Basic access control gaps

These are valuable but increasingly expected. They are found by automated
scans before the audit even begins. Bug bounty programs reward them at
Medium to Low severity in most cases.

### Where the Real Money Is

Research analyzing over $1B of real-world DeFi losses (arXiv:2507.20175)
found that the majority of catastrophic exploits traced back to:

- Protocol logic design flaws
- Broken invariants that "bug-free" code violated
- Business logic that worked in isolation but failed under composition

The Euler Finance exploit ($196M) is the canonical example. The code had
no reentrancy, no overflow, no oracle manipulation. The access control was
correct. But the solvency invariant could be broken through a specific
sequence of valid operations. Business logic. Not a bug. A broken guarantee.

### The Two Questions Every Auditor Must Ask

When reading ANY function in ANY contract, ask BOTH of these:

```
QUESTION 1: "What does this contract do?"
  → Describes features
  → Finds implementation bugs
  → Table stakes for any auditor

QUESTION 2: "If this invariant breaks, can someone:
             STEAL money?
             PRINT money from nothing?
             FREEZE money permanently?
             BRICK the protocol?"
  → Describes impact
  → Finds business logic flaws
  → This is where $10k–$50k+ findings live
```

If Question 2 has a "yes" answer for any function, that function is your
highest-priority audit target. That is what goes into your design_doc.md
as a formal property.

### Priority Order for Audit Time Allocation

Spend your time in this order. These categories cover the majority of
catastrophic losses in modern DeFi:

```
Priority 1: Asset Conservation
            Can value be created from nothing?
            → Highest frequency of critical findings

Priority 2: Solvency
            Can liabilities exceed assets?
            → Highest dollar impact per exploit (Euler: $196M)

Priority 3: Share / Accounting Integrity
            Can share price be manipulated?
            → Common in ERC4626, vaults, LSTs, LRTs

Priority 4: State Machine Correctness
            Can operations happen in wrong order?
            → Often invisible to classical auditors

Priority 5: Cross-Contract Invariant Preservation
            Do multi-contract systems stay synchronized?
            → Missed by single-contract analysis tools

Priority 6: Access Control
            Can unauthorized actors act?
            → Still important but more often caught automatically

Priority 7: Fund Lockup Prevention
            Can funds be permanently frozen?
            → Often overlooked, high impact when found
```

### Severity Framework (Immunefi / Sherlock / HackenProof)

| Severity | Value | Trigger |
|----------|-------|---------|
| CRITICAL | $10k–$50k+ | Theft, permanent freezing, insolvency, unauthorized minting |
| HIGH | $5k–$20k | Temp freeze >1 month, yield theft, broken accounting |
| MEDIUM | $1k–$5k | DoS >1 week, state inconsistency |
| LOW | <$1k | Gas, informational |

### Realistic Expectations

Starting out (months 1–3): $500–$3k per valid finding
Growing (months 3–12): $2k–$10k per finding
Established: $5k–$50k+ per finding on complex protocols

---

## PART 4 (UPDATED): THE 10 BUSINESS LOGIC BUG CATEGORIES
### CLAIM → PROPERTY: a useful convention for design_doc.md, not a required format

These are not theoretical. Each category maps to real exploits that cost protocols millions.

**Note on format:** AI Composer doesn't parse or require any particular syntax in your design doc — the whole
document is read as prose by an LLM classification step that derives structured requirements from it, so what
matters is writing a clear, unambiguous, standalone claim, not matching a specific template. The `PROPERTY:`
labeling below is a readability convention worth adopting anyway (it keeps a long design doc organized and makes
each claim easy to spot on review), but there's nothing special about the literal string — a plain paragraph
saying the same thing works identically well.

---

### Category 1: Asset Conservation

**The Core Claim:**
```
Assets cannot be created from nothing.
```

**Examples of violations:**
- Inflation bugs (totalSupply increases without mint)
- Double-mint bugs (same value minted twice)
- Share inflation (share count inflates without deposit)
- Accounting mismatch (tracked value diverges from real value)

**Properties to include in design_doc.md:**
```
PROPERTY:
Total assets accounted for by the protocol must never exceed
actual assets held by the protocol.

PROPERTY:
A user cannot increase their claim on protocol assets
without providing equivalent value.
```

**The Question to Ask:**
"Is there any sequence of valid function calls that increases
total claimable value without real assets entering the protocol?"

---

### Category 2: Funds Cannot Be Withdrawn Twice

**The Core Claim:**
```
The same value cannot be redeemed twice.
```

**Examples of violations:**
- Withdrawal accounting bugs (claim not decremented)
- Claim tracking bugs (per-user record not updated)
- Reward claim bugs (double-claim in same block)

**Properties to include in design_doc.md:**
```
PROPERTY:
After assets are withdrawn, the user's claim must
decrease by exactly the withdrawn amount.

PROPERTY:
A second withdrawal by the same user for the same
assets must revert.
```

**The Question to Ask:**
"Is the user's claim decremented BEFORE or AFTER the transfer executes?
If AFTER, reentrancy or state manipulation can double-spend."

---

### Category 3: Reserved Funds Are Untouchable

**The Core Claim:**
```
Reserved funds cannot be accessed through any path
not intended for them.
```

**Examples of violations:**
- Admin withdrawal that counts reserved funds as available
- Emergency functions that bypass reservation checks
- Rounding that allows small amounts to leak from reserved pool

**Properties to include in design_doc.md:**
```
PROPERTY:
Reserved funds cannot be withdrawn by any actor,
including the owner or admin.

PROPERTY:
Withdrawals cannot reduce the reserved balance.
The invariant availableBalance = contract.balance - reservedBalance
must hold after every call to withdraw().
```

**The Question to Ask:**
"Is there any function — including owner-only, emergency, or recovery
functions — that can reduce the reserved balance without going
through the intended reserve mechanism?"

---

### Category 4: Solvency

**The Core Claim:**
```
Protocol remains fully collateralized.
Total liabilities must never exceed total assets.
```

**Real-world impact:**
Euler Finance ($196M) — broken solvency through valid operations.
Lending protocols across DeFi — broken LTV after price movement.

**Properties to include in design_doc.md:**
```
PROPERTY:
Total liabilities must never exceed total assets.
At all times: sum(what protocol owes) ≤ sum(what protocol holds).

PROPERTY:
A user cannot borrow beyond their allowed collateral.
The protocol's LTV ratio must be enforced on every borrow call.
```

**The Question to Ask:**
"What does this protocol OWE to its users?
What does it HOLD as real assets?
Under what sequence of operations can OWED > HELD?"

---

### Category 5: Exchange Rate / Share Price Integrity

**The Core Claim:**
*(Critical for ERC4626, vaults, LSTs, LRTs, yield vaults)*
```
Share value cannot be artificially inflated.
Share issuance must be proportional to assets deposited.
```

**Real-world impact:**
Donation-style attacks (Euler, multiple ERC4626 vaults) manipulate
exchange rates and cascade into catastrophic borrowing exploits.

**Properties to include in design_doc.md:**
```
PROPERTY:
External ETH or token transfers to the contract must not
artificially inflate the share price.

PROPERTY:
Share issuance must be proportional to assets deposited.
A deposit of X assets must yield Y shares where Y is computed
consistently before and after the deposit.

PROPERTY:
The first depositor cannot manipulate the initial share price
to disadvantage subsequent depositors.
```

**The Question to Ask:**
"What happens if someone sends ETH or tokens directly to the contract
(not through the deposit function) before anyone else deposits?
Does this inflate the share price for the first depositor?"

---

### Category 6: Access Control Guarantees

**The Core Claim:**
```
Only authorized actors can perform privileged actions.
```

**Properties to include in design_doc.md:**
```
PROPERTY:
Only the owner address may call privileged administrative
functions. Any other caller must revert.

PROPERTY:
Only authorized actors may call restricted operations.
Any other caller must revert.

PROPERTY:
No unauthorized actor can move protocol funds, regardless of
the sequence of calls made.
```

**The Question to Ask:**
"Is the access check at the TOP of the function, before any state changes?
Or is there a path through the function that bypasses the check?"

---

### Category 7: State Machine Correctness

**The Core Claim:**
```
Actions must happen in a valid sequence.
The protocol must enforce operation ordering.
```

**Example of a state machine violation:**
```
Correct sequence:
  deposit() → lock() → finalizeOperation()

Must never be possible:
  finalizeOperation() without prior lock()
  (would use unlocked funds, bypassing the lock requirement)
```

**Properties to include in design_doc.md:**
```
PROPERTY:
Critical operations may only occur from the correct prior state.
A call to finalizeOperation() must require that funds are locked,
not just that they exist in the contract.

PROPERTY:
The locked state and the finalized state must transition
atomically. There must be no intermediate state where funds
are neither locked nor finalized.
```

**The Question to Ask:**
"Draw the valid state transitions as a flowchart.
Can a caller reach a final state by skipping one of the required
intermediate steps? What is the impact if they can?"

---

### Category 8: Accounting Synchronization

**The Core Claim:**
*(Very common in multi-contract systems)*
```
All accounting systems tracking the same value
must stay synchronized at all times.
```

**Example of the problem:**
```
Vault balance shows: 1000 ETH
Share accounting shows: 900 ETH equivalent

Where is the 100 ETH? Who can claim it?
Can someone withdraw the discrepancy?
```

**Properties to include in design_doc.md:**
```
PROPERTY:
Whenever assets enter the protocol, total accounted assets
must increase by exactly the same amount.

PROPERTY:
Whenever assets leave the protocol, total accounted assets
must decrease by exactly the same amount.

PROPERTY:
At all times: actual_balance == sum(all_user_claims) + protocol_reserves.
No assets should be unaccounted for.
```

**The Question to Ask:**
"Are there two or more variables that both track the same underlying value?
Is there any function that updates one but not the other?
What can an attacker do with the gap?"

---

### Category 9: Reward Integrity

**The Core Claim:**
```
Rewards are distributed fairly.
Each reward can only be claimed once.
```

**Properties to include in design_doc.md:**
```
PROPERTY:
A user cannot claim rewards twice for the same accrual period.
After claimRewards() succeeds, the user's claimable amount
must be zero.

PROPERTY:
Total rewards claimed across all users cannot exceed
total rewards accrued by the protocol.
```

**The Question to Ask:**
"Is the user's claimable reward amount set to zero BEFORE
the token transfer executes?
If the token has a callback (ERC777, ERC1155), can the user
re-enter during the transfer and claim again?"

---

### Category 10: Funds Cannot Become Permanently Locked

**The Core Claim:**
*(Often overlooked, high impact when found)*
```
Protocol funds remain recoverable.
Valid withdrawal paths always exist.
Pause cannot permanently lock user funds.
```

**Properties to include in design_doc.md:**
```
PROPERTY:
Valid withdrawal paths must always exist for users
with non-zero balances.

PROPERTY:
The pause mechanism is a temporary emergency measure.
It must not permanently prevent fund recovery.
At minimum, an emergency withdrawal path must remain
accessible even while the contract is paused.
```

**The Question to Ask:**
"If the contract enters its paused/emergency state permanently,
can users still get their funds out?
Is there ANY scenario where both the normal AND emergency
withdrawal paths fail simultaneously?"

---

## THE AUDITOR MINDSET (Add to Every Design Doc Session)

Before writing any property, run this mental filter:

```
FOR EVERY INVARIANT YOU FIND, ASK:

"If this invariant breaks..."

→ Can someone STEAL money?
  YES → CRITICAL finding. Write this property FIRST.

→ Can someone PRINT money from nothing?
  YES → CRITICAL finding. Asset conservation violation.

→ Can someone FREEZE money permanently?
  YES → CRITICAL/HIGH finding. Lockup violation.

→ Can someone BRICK the protocol entirely?
  YES → HIGH/CRITICAL finding. State machine or governance violation.

→ Can someone gain UNFAIR ADVANTAGE?
  YES → HIGH finding. Accounting or reward integrity violation.

→ None of the above?
  → LOW/MEDIUM or informational. Not your priority.
```

This filter determines which properties go into your design_doc.md.
Only properties that produce a "yes" to one of these questions are
worth formalizing and verifying with Certora.

---

## PART 6, PROMPT A (UPDATED): 10-CATEGORY BUSINESS LOGIC ANALYSIS

*Use this exact prompt with GitHub Copilot Pro or Claude on any contract.*
*This replaces the previous generic version.*

```
Analyze this smart contract for business logic vulnerabilities.
This is for a bug bounty submission targeting $10k–$50k+ findings.

Ignore classical bugs like reentrancy and overflow — assume those are
handled by automated tools. Focus entirely on business logic:
invariants whose violation leads to theft, inflation, freezing, or
protocol shutdown.

CONTRACT CODE:
[PASTE FULL CONTRACT]

ANALYZE EACH OF THESE 10 CATEGORIES:

────────────────────────────────────────────────────────
1. ASSET CONSERVATION
   Claim: Assets cannot be created from nothing.

   Check:
   - Can total claimable value exceed real assets held?
   - Can a user increase their claim without depositing value?
   - Does totalSupply ever increase without an explicit mint call?
   - Are there rounding paths that create dust value over time?

   If violated: state the EXACT function, the EXACT sequence
   of calls, and the dollar impact.
   Also state the plain-English property for design_doc.md.

────────────────────────────────────────────────────────
2. FUNDS CANNOT BE WITHDRAWN TWICE
   Claim: The same value cannot be redeemed twice.

   Check:
   - Is the user's claim decremented BEFORE or AFTER the transfer?
   - Can withdraw() complete and then succeed again immediately?
   - Is there any path where claim is not cleared but funds are sent?

────────────────────────────────────────────────────────
3. RESERVED FUNDS ARE UNTOUCHABLE
   Claim: Designated reserved funds cannot be accessed through
   any path not intended for them.

   Check:
   - What funds are staged, reserved, or locked for specific use?
   - Can the owner, admin, or any actor access them through
     withdraw(), emergencyWithdraw(), or any other function?
   - Does availableBalance correctly exclude reserved amounts?

────────────────────────────────────────────────────────
4. SOLVENCY
   Claim: Protocol remains fully collateralized.
   Total liabilities must never exceed total assets.

   Check:
   - What does this protocol OWE to users? (list it explicitly)
   - What does this protocol HOLD in real assets? (list it)
   - Under what sequence of valid operations can OWED > HELD?

────────────────────────────────────────────────────────
5. EXCHANGE RATE / SHARE PRICE INTEGRITY
   Claim: Share value cannot be artificially inflated.
   Share issuance is proportional to assets deposited.

   Check:
   - What happens if ETH/tokens are sent directly to the contract?
   - Can a first depositor attack manipulate the initial price?
   - Is the share price formula stable at extreme reserve ratios?

────────────────────────────────────────────────────────
6. ACCESS CONTROL GUARANTEES
   Claim: Only authorized actors can perform privileged actions.

   Check:
   - List every function that changes critical state
   - For each: where exactly is the access check? Is it first?
   - Can any check be bypassed through any code path?

────────────────────────────────────────────────────────
7. STATE MACHINE CORRECTNESS
   Claim: Actions must happen in a valid sequence.

   Check:
   - Draw the valid state transitions
   - Can any required step be skipped?
   - Can the contract reach a state through invalid ordering?

────────────────────────────────────────────────────────
8. ACCOUNTING SYNCHRONIZATION
   Claim: All accounting systems tracking the same value
   stay synchronized.

   Check:
   - List all variables that track the same underlying value
   - Is there any function that updates one but not the other?
   - What can an attacker do with any gap between them?

────────────────────────────────────────────────────────
9. REWARD INTEGRITY
   Claim: Rewards are distributed fairly.
   Each reward can only be claimed once.

   Check:
   - Is claimable amount zeroed BEFORE the transfer?
   - What if claimRewards() is called twice in the same block?
   - Can a token callback enable double-claim?

────────────────────────────────────────────────────────
10. FUNDS CANNOT BECOME PERMANENTLY LOCKED
    Claim: Valid withdrawal paths always exist.
    Pause cannot permanently lock user funds.

    Check:
    - If the contract is paused indefinitely, can users withdraw?
    - Is there an emergency path? Is it always accessible?
    - Can both normal AND emergency paths fail simultaneously?

────────────────────────────────────────────────────────

FOR EACH CATEGORY PROVIDE:

VULNERABILITY FOUND: YES / NO

If YES:
- EXACT FUNCTION: [function name]
- EXPLOIT SEQUENCE: [step-by-step — what the attacker does]
- FINANCIAL IMPACT: [$ estimate]
- AUDITOR MINDSET CHECK: does this let someone STEAL / PRINT /
  FREEZE / BRICK? → determines severity
- DESIGN DOC PROPERTY: [exact plain-English property statement
  to add to design_doc.md]
- SUGGESTED FIX: [brief code change]

If NO:
- ONE SENTENCE explaining why the category is safe here
```

---

## PART 13 (UPDATED): CATASTROPHIC FAILURE SCENARIOS
### A recommended section for every design_doc.md — not a format AI Composer recognizes structurally

Add this section to EVERY design_doc.md. This forces you to name the five worst outcomes and what prevents each
one. **Note on mechanism:** there's no code in AI Composer that looks for a section literally titled "Catastrophic
Failure Scenarios," a `CF-N` numbering scheme, or the `Must Never Happen:`/`Affected Functions:` labels below —
the design doc is read as prose by an upstream classification step (not string-matched by heading), so nothing
about this specific structure gets special treatment over any other clearly-written part of the document. The
value of doing this exercise is in the thinking it forces (naming the worst outcomes explicitly tends to surface
requirements you'd otherwise miss), and clear, explicit language here is more likely to be picked up accurately
regardless of the heading style you use.

```markdown
# Catastrophic Failure Scenarios

## CF-1: Theft of Reserved Funds

Impact:
Loss of reserved protocol funds. Core operations permanently broken.
Estimated loss: [$ amount based on TVL]

Must Never Happen:
Any withdrawal or transfer reduces the reserved balance.
Any actor accesses reserved funds through non-intended paths.
availableBalance() returns a value that includes reserved funds.

Affected Functions:
withdraw()
emergencyWithdraw()
transfer()
[any function that moves funds]

What AIComposer should verify:
After any call to withdraw(), reservedBalance is unchanged.
availableBalance() always equals contract.balance minus reservedBalance.
Reserved funds can only be reduced through their intended mechanism.

---

## CF-2: Protocol Insolvency

Impact:
User claims exceed protocol assets. Protocol cannot honor withdrawals.
Estimated loss: [$ amount — potentially entire TVL]

Must Never Happen:
Total liabilities (what the protocol owes) exceeds
total assets (what the protocol holds).

Affected Functions:
deposit(), withdraw(), borrow(), repay(), liquidate(), getHealthFactor()

What AIComposer should verify:
At all times: sum(user_claims) ≤ address(this).balance + external_assets.
No sequence of valid operations makes the protocol insolvent.

---

## CF-3: Share / Accounting Inflation

Impact:
Attacker obtains disproportionate ownership of the protocol.
Subsequent depositors receive far fewer shares than they should.
Estimated loss: [$ amount per attack]

Must Never Happen:
Shares minted to a user exceed the value of assets they deposited.
Share price changes as a result of external transfers, not deposits.
First depositor can set an initial share price that harms others.

Affected Functions:
deposit(), mint(), redeem(), withdraw(), convertToShares(), convertToAssets()

What AIComposer should verify:
Share price remains stable before and after any deposit.
Shares minted always proportional to assets received.
External ETH transfers do not change the share price calculation.

---

## CF-4: Permanent Lockup

Impact:
User funds become permanently unrecoverable.
Protocol effectively shut down.
Estimated loss: [$ amount — all user deposits]

Must Never Happen:
A state exists where withdraw() fails AND emergencyWithdraw() fails.
The pause mechanism eliminates all withdrawal paths.
Owner can freeze the contract with no recovery mechanism.

Affected Functions:
withdraw(), emergencyWithdraw(), pause(), unpause()

What AIComposer should verify:
For any user with balance > 0, at least one withdrawal path succeeds.
Pause does not block ALL fund recovery paths simultaneously.

---

## CF-5: Unauthorized Control

Impact:
Attacker gains privileged authority over the protocol.
Can drain treasury, change parameters, or halt operations.
Estimated loss: [$ amount — entire protocol TVL at risk]

Must Never Happen:
Non-owner successfully calls any owner-only function.
Ownership transfers to an unauthorized address.
Admin privileges are escalated without explicit owner action.

Affected Functions:
pause(), unpause(), setFees(), addAuthorizedDepositor(),
transferOwnership(), upgradeTo()

What AIComposer should verify:
Every privileged function reverts for non-authorized callers.
Ownership cannot change without explicit action from current owner.
```

---

## HOW THIS CHANGES YOUR DESIGN_DOC.MD

The properties you write in design_doc.md should now follow this pattern
for every single entry:

```
PROPERTY NAME: [descriptive name]

Claim: [one sentence — what the protocol promises]

If violated: [STEAL / PRINT / FREEZE / BRICK] → severity [CRITICAL/HIGH]

Statement:
[exact plain-English description of what must always be true]

What breaks if this fails:
[specific attack, dollar impact, who is affected]

Priority: [1-7 based on the priority order in Part 3]
```

**Example using a vault contract:**

```
PROPERTY NAME: Reserved Fund Protection

Claim: Reserved funds are exclusively for their intended purpose.

If violated: STEAL → CRITICAL ($50k+ bounty)

Statement:
The reserved balance can only decrease as a result of a successful
call to the intended reserve mechanism. No other function — including
withdraw(), emergencyWithdraw(), or any admin-only function —
may reduce the reserved balance.

What breaks if this fails:
An admin or attacker calls withdraw() to drain funds that were
reserved for protocol operations. The critical operation fails
or succeeds with wrong parameters. Up to the full reserved
amount is permanently lost from the protocol.

Priority: 1 (Asset Conservation — reserved funds variant)
```

---

*Business Logic Core — June 2026*
*Foundation: arXiv:2507.20175*
*These sections replace Parts 3, 4, 6 Prompt A, and 13 of MASTER_AUDIT_GUIDE_V2.md*