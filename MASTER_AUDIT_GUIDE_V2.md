# Master Audit Guide V2: GitHub Copilot Pro + AIComposer
## Smart Contract Bug Bounty — Everything in One Place

> This guide incorporates everything from our conversation + the 5 uploaded docs.
> 18 items from our conversation were missing from the docs — all added here.
> All 10 problems from the previous docs are fixed.

---

## TABLE OF CONTENTS

1.  The Correct Order — How Everything Connects
2.  Command Reference — 4 Tools, Not Interchangeable
3.  Impact-Driven Paradigm + Severity Framework
4.  The 10 High-Value Business Logic Bug Categories
5.  Phase A: 7-Step Property Extraction (GitHub Copilot Pro Prompts)
6.  Phase A: Business Logic Manual Analysis (Copilot Prompts)
7.  Phase B: AIComposer Pipeline (After AutoSetup)
8.  Counter-Example Analysis (cex-analyzer)
9.  Report Writing — Complete Template
10. Finding Classification — 4 Types
11. PoC Test Format — Foundry
12. Contest Submission Strategy
13. Catastrophic Failure Scenarios Template
14. Complete Worked Example
15. WSL Ubuntu Setup Reference
16. Quick Reference Checklist

---

## PART 1: THE CORRECT ORDER — HOW EVERYTHING CONNECTS

```
THE WINNING FORMULA (From Our Conversation)
════════════════════════════════════════════

Design Doc (What protocol claims it does)
           ↓
GitHub Copilot Pro (Helps you extract + write)
           ↓
AIComposer (Extracts properties → generates CVL automatically)
           ↓
Certora Prover (Finds violations mathematically)
           ↓
cex-analyzer (Explains violations in plain English)
           ↓
Contest Report (Documented → Formal → Practical)
           ↓
VALIDATED FINDINGS ✓ (Judges cannot invalidate what the
                       protocol itself promised)

════════════════════════════════════════════
PHASE A — RIGHT NOW (No AutoSetup needed)
────────────────────────────────────────────
You + GitHub Copilot Pro
  → 7-step property extraction
  → Business logic manual analysis
  → design_doc.md (2000+ words, plain English only)
  → PoC test drafts

PHASE B — AFTER AUTOSETUP (Certora access granted)
────────────────────────────────────────────────────
Your design_doc.md → AIComposer phases 0-5
  → CVL specs generated automatically
  → Prover finds violations
  → cex_analyzer explains them
  → You write contest report
```

---

## PART 2: COMMAND REFERENCE — 4 TOOLS, NOT INTERCHANGEABLE

| Command | What It Does | When to Use |
|---------|-------------|-------------|
| `tui-autoprove` (installed) or `python -m composer.cli.tui_autoprove` | Full audit pipeline with live TUI dashboard | PRIMARY — use this for all audits |
| `console-autoprove` (installed) or `python -m composer.cli.console_autoprove` | Same pipeline, terminal output | When TUI has rendering issues, or for CI/logging |
| `certoraRun contract.conf` | Runs Certora Prover on a manually-written .conf | After you write CVL specs manually |

**Standard audit command:**
```bash
tui-autoprove \
  ~/projects/protocol \
  src/Contract.sol:ContractName \
  design_doc.md \
  --cloud \
  --cache-ns my-audit-v1 \
  --max-concurrent 4
```

**Critical clarification:** `--cloud` only means the Certora Prover runs on
Certora's cloud servers instead of your local machine. It does NOT bypass
the AutoSetup requirement. Phase 1 still needs AutoSetup regardless.

---

## PART 3: IMPACT-DRIVEN PARADIGM + SEVERITY FRAMEWORK

### Old Thinking vs New Thinking

```
OLD (Generic bug hunting)          NEW (Impact-driven hunting)
─────────────────────────          ─────────────────────────────
Find: reentrancy                   Hunt for: THEFT (direct fund loss)
Find: overflow                     Hunt for: FREEZING (locked funds)
Find: oracle manipulation          Hunt for: MINTING (create from nothing)
Find: access control gaps          Hunt for: INSOLVENCY (liab > assets)

Ask: "What's wrong with code?"     Ask: "What can BREAK the protocol?"
Value: $5k–$20k                    Value: $10k–$50k+
Result: Known patterns             Result: Business logic flaws
```

### Key Insight from Our Conversation

A protocol can be COMPLETELY BUG-FREE — no reentrancy, no overflow, perfect
access control — and still have devastating business logic vulnerabilities.
This is why we hunt IMPACT, not just bugs.

### Severity Framework (Immunefi / Sherlock / HackenProof)

| Severity | Value | Requirements (ANY of these) |
|----------|-------|---------------------------|
| CRITICAL | $10k–$50k+ | Direct theft of funds, permanent freezing, protocol insolvency, unauthorized minting, governance manipulation |
| HIGH | $5k–$20k | Temp freezing >1 month, unclaimed yield theft, broken accounting, DoS >7 days |
| MEDIUM | $1k–$5k | DoS >1 week, state inconsistency, broken returns |
| LOW | <$1k | Gas optimizations, informational, negligible leaks |

### Realistic Earning Expectations

Starting out (first 3 months): $500–$3k per valid finding
Growing (3–12 months): $2k–$10k per finding
Established (12+ months): $5k–$50k+ per finding

### What AIComposer's Property Analysis Actually Extracts

Based on the actual system prompts (`property_analysis_system_prompt.j2`
and `property_analysis_prompt.j2`), here is exactly what AIComposer's
auto-prove pipeline looks for and what it avoids:

```
AIComposer EXTRACTS:                          AIComposer AVOIDS:
  ✓ Invariants (always-true statements)        ✗ Off-chain events
  ✓ Safety properties (expected behavior)      ✗ Hash collisions / preimage
  ✓ Attack vectors (plausible exploits)        ✗ Event emission
  ✓ Rounding discrepancies                     ✗ Overflow (checked arithmetic
  ✓ Oracle / price manipulation                 covers this)
  ✓ MEV / front-running                        ✗ Type-implied (uint256 ≥ 0)
  ✓ Access-control bypasses                    ✗ Overly broad statements
  ✓ Lifecycle / pause bugs                     ✗ Infeasible attacks (EVM
  ✓ Governance abuse                            consensus compromise)
  ✓ State machine violations                   ✗ Padding / duplicate properties
  ✓ Accounting desynchronization               ✗ Trivial corollaries
```

**The multi-round convergence rule** (directly from the system prompt):
*"Return an empty list rather than pad with low-value properties. The
convergence signal — an empty items list with a reasoning field that
explains what you looked at and why nothing new emerged — is the
desired outcome, not a failure."*

**The reasoning field rule** (directly from the system prompt):
*"Your reasoning field is load-bearing. It is read by future-round
agents and the human reviewer. Be specific about what you looked at —
name functions, name parameters, name attack patterns."*

This is exactly what your manual claims extraction (Step 2) and property
conversion (Step 4) should mirror: be specific, tie everything to
documented claims, and don't invent properties just to fill space.

---

## PART 4: THE 10 HIGH-VALUE BUSINESS LOGIC CATEGORIES

> Note: Previous docs incorrectly said "8 categories" in 14 places. There are 10.
> Authoritative names from BUSINESS_LOGIC_CORE.md — use these exact names in design_doc.md.

| # | Category | Attack Type | Severity |
|---|----------|------------|---------|
| 1 | Asset Conservation | Create value from nothing | CRITICAL |
| 2 | Funds Cannot Be Withdrawn Twice | Spend same value twice | CRITICAL |
| 3 | Reserved Funds Are Untouchable | Steal reserved/staged funds | CRITICAL |
| 4 | Solvency | Make liabilities exceed assets | CRITICAL |
| 5 | Exchange Rate / Share Price Integrity | Inflate share value via donation | CRITICAL |
| 6 | Access Control Guarantees | Call privileged function without auth | CRITICAL |
| 7 | State Machine Correctness | Skip required steps/transitions | HIGH |
| 8 | Accounting Synchronization | Desync multi-contract accounting | HIGH/CRITICAL |
| 9 | Reward Integrity | Double-claim rewards | HIGH |
| 10 | Funds Cannot Become Permanently Locked | Permanently lock user funds | HIGH/CRITICAL |

### What to Ask for Each Category

```
1. Asset Conservation:     Can total assets increase without a corresponding deposit?
2. Double Withdrawal:      Is the claim decremented BEFORE or AFTER the transfer?
3. Reserved Funds:         Can owner/admin access funds reserved for specific operations?
4. Solvency:               What does the protocol OWE? What does it HOLD? Is HOLD ≥ OWE?
5. Share Price:            Can I donate ETH directly to inflate share price before others deposit?
6. Access Control:         Does every privileged function check msg.sender early in execution?
7. State Machine:          Can a function be called in the wrong order or sequence?
8. Accounting Sync:        Are there two variables tracking the same value that could diverge?
9. Reward Integrity:       What happens if claimRewards() is called twice in the same block?
10. Fund Lockup:           If the contract is paused, can users STILL access their funds?
```

---

## PART 5: PHASE A — 7-STEP PROPERTY EXTRACTION
### GitHub Copilot Pro Prompts (All Exact, Ready to Use)

**Correct order — this is important:**
```
Step 1: Understand → Step 2: Extract Claims → Step 3: Map Components
Step 4: Convert to Properties → Step 5: Pre/Postconditions
Step 6: Draft design_doc.md → Step 7: Review & Refine
```

---

### STEP 1: Understand the Contract
**Tool:** GitHub Copilot Pro Chat
**Who does it:** Copilot (you read the output)

**Exact Prompt:**
```
@workspace I need to audit this smart contract for a bug bounty submission.
Please analyze it and provide:

1. SUMMARY: What does this contract do in 3–4 sentences?
2. STATE VARIABLES: List every state variable with its type and purpose
3. FUNCTIONS: List every function with visibility, mutability, and a
   one-line description of what it does
4. ACTORS: Who are the key actors? (owner, admin, user, etc.)
   What can each actor do?
5. MODIFIERS: List all modifiers and what they enforce
6. EXTERNAL DEPENDENCIES: What external contracts does this call?
7. EVENTS: List all events and when they are emitted

Format clearly with a header for each section.
```

**What you do with the output:** Read carefully. Ask follow-up questions
about anything you do not fully understand before moving to Step 2.

---

### STEP 2: Extract Documented Claims
**Tool:** You read the code, Copilot validates
**Who does it:** YOU first (critical thinking), then Copilot

**What you do MANUALLY:**
Read the contract and extract every explicit promise. Look for:
- NatSpec @notice and @dev comments
- require() error messages (reveal what SHOULD be true)
- Function names (onlyOwner, whenNotPaused = claims in themselves)
- Variable names (reservedBalance, availableBalance = claims about fund separation)
- Modifiers (each one is a claim about who can call what)

**Exact Prompt (after your own reading):**
```
Based on this smart contract's code comments, NatSpec documentation,
require() statements, error messages, function names, and modifier names,
extract ALL explicit and implicit claims the protocol makes about its
own behavior.

For each claim:
CLAIM: [what the protocol promises]
SOURCE: [exact location — function name, line, or NatSpec comment]
TYPE: [access control / accounting / fund protection / state transition / other]

Be exhaustive. Even function names like "onlyOperator" are claims.
Even require(amount > 0, "Zero amount") is a claim.
```

---

### STEP 3: Identify Core System Components
**Tool:** GitHub Copilot Pro Chat
**Who does it:** Copilot (you verify accuracy)

**Exact Prompt:**
```
Analyze this smart contract and identify 3–6 major functional components
or areas of responsibility.

For each component provide:
- COMPONENT NAME: (descriptive, not just the contract name)
- PURPOSE: What does this component do?
- STATE IT MANAGES: Which state variables belong to this component?
- FUNCTIONS: Which functions implement this component?
- EXTERNAL DEPENDENCIES: What external contracts does it interact with?
- RISK AREA: Where could this component have security issues?

Format as a separate section for each component.
```

---

### STEP 4: Convert Claims to Formal Properties
**Tool:** YOUR thinking, Copilot validates
**Who does it:** YOU (most critical step)

**Your job:** Take each claim from Step 2 and write it as a testable
mathematical statement. Use this format:

```
CLAIM:    "Only designated operator can perform critical operations"
PROPERTY: finalizeOperation() → msg.sender must equal operator address

CLAIM:    "Reserved balance cannot be withdrawn"
PROPERTY: withdraw(amount) requires amount ≤ (address(this).balance − reservedBalance)
          AND reservedBalance does not decrease as result of withdraw()

CLAIM:    "Pausing prevents all deposits"
PROPERTY: depositsPaused == true → deposit() reverts
          depositsPaused == true → addLiquidity() reverts
```

**Exact Prompt (to validate your translations):**
```
I have translated these documented claims from the contract into
formal properties. Please review each one:

[PASTE YOUR CLAIM → PROPERTY TRANSLATIONS]

For each property tell me:
1. Is my translation accurate and complete?
2. Are there edge cases or conditions I missed?
3. Could this property be violated in any way I haven't considered?
4. Is the property too narrow? (Missing related cases?)
5. Is the property too broad? (Would catch false positives?)
```

---

### STEP 5: Define Pre/Postconditions for Critical Functions
**Tool:** Copilot Pro (you specify which functions)
**Who does it:** You + Copilot

**Exact Prompt:**
```
For each of these critical functions in the contract, define complete
pre/postconditions. Focus on security-relevant conditions only.

Functions to analyze:
- [function 1]
- [function 2]
- [function 3]

For EACH function provide:

PRECONDITIONS (must be true BEFORE the function can execute):
- permission check: [who is allowed to call this?]
- state check: [what contract state must be true?]
- input check: [what must be true about the parameters?]
- balance check: [what balance conditions must hold?]

POSTCONDITIONS (must be true AFTER successful execution):
- state changes: [exactly what state variables changed and how?]
- balance changes: [what balances changed and by how much?]
- events: [which events are emitted?]
- what did NOT change: [what state must remain unchanged?]

REVERTS IF:
- [list every condition that should cause a revert]

INVARIANTS THAT MUST HOLD THROUGHOUT:
- [what must remain true even during execution?]
```

---

### STEP 6: Draft design_doc.md
**Tool:** Copilot Pro (structures and writes)
**Who does it:** Copilot + You (you provide all content from Steps 1–5)

**CRITICAL RULE: design_doc.md = PLAIN ENGLISH ONLY**
- NO Solidity code
- NO CVL code (invariant, rule, etc.)
- NO pseudocode
- Only clear prose and structured lists

AIComposer converts your English into CVL automatically.
Putting CVL in the doc confuses the LLM and reduces quality.

**Exact Prompt:**
```
Help me write a design document for AIComposer formal verification
using the content I have gathered.

STRICT RULE: The document must contain ONLY plain English.
No Solidity code, no CVL syntax, no pseudocode.
If I have included any code, convert it to plain English descriptions.

Use this exact 9-section structure:

1. Executive Summary (2–3 sentences only)
2. Core System Components (one subsection per component from Step 3)
3. Documented Claims (all claims from Step 2, organized by type)
4. Formal Properties (translations from Step 4, plain English only)
5. Critical Function Specifications (pre/postconditions from Step 5)
6. High-Level Properties for Verification (6–10 most important)
7. Edge Cases and State Transitions
8. Business Logic Concerns (all 10 categories addressed for this protocol)
9. Assumptions

Here is my content to organize:
[PASTE ALL YOUR FINDINGS FROM STEPS 1–5]

Make each property specific to THIS protocol, not generic.
Target length: 2000–5000 words.

Example of GENERIC (bad — could apply to ANY protocol):
  "Users cannot withdraw more than they deposited."

Example of SPECIFIC (good — tied to THIS protocol's actual logic):
  "When withdraw() is called, the amount available to withdraw is
   calculated as (userShares * (address(this).balance - reservedBalance))
   / totalSupply, not as (userShares * address(this).balance) / totalSupply.
   This ensures reserved funds are excluded from the withdrawable pool."
```

---

### STEP 7: Review and Refine
**Tool:** Copilot Pro (quality check)
**Who does it:** You first (self-review), then Copilot

**Self-Review Questions:**
- Does every property tie to a documented claim? (no invented properties)
- Is every property specific? (not "fees work correctly")
- Are all 10 business logic categories addressed?
- Are edge cases covered? (zero amounts, empty states, pause states)
- Does the doc contain ANY code? (remove it if so)
- Would a judge accept each finding based on this documentation?

**Exact Prompt:**
```
Review this design_doc.md I have written for AIComposer formal verification.

[PASTE YOUR COMPLETE design_doc.md]

Check for these specific issues:

1. SPECIFICITY: Are properties specific to this protocol or are they generic?
   Flag any property that could apply to ANY protocol unchanged.

2. COMPLETENESS: Are all 10 business logic categories addressed?
   (asset conservation, double withdrawal, reserved funds, solvency,
   share price integrity, access control, state machine correctness,
   accounting synchronization, reward integrity, fund lockup prevention)

3. TRACEABILITY: Does each property trace back to a documented claim?
   Flag any property I invented without a documented source.

4. CODE PRESENCE: Does the document contain any CVL, Solidity, or pseudocode?
   Convert anything found to plain English.

5. EDGE CASES: Are these edge cases covered?
   - Zero amount inputs
   - Maximum amount inputs
   - Empty/zero state (empty pool, zero balance)
   - Paused state operations
   - Transition states (mid-operation)

6. IMPACT CLARITY: For each property, is it clear what breaks if it fails?
   (theft / freezing / insolvency / minting / none)

Give specific line-by-line feedback on what to improve.
```

---

## PART 6: PHASE A — BUSINESS LOGIC MANUAL ANALYSIS
### Replaces AIComposer Phase 4 While Waiting for AutoSetup

---

### Prompt A: Full 10-Category Analysis

```
Analyze this smart contract for ALL 10 business logic bug categories.
This is for a bug bounty submission. Be specific and thorough.

CONTRACT CODE:
[PASTE FULL CONTRACT]

ANALYZE EACH CATEGORY:

1. ASSET CONSERVATION
   Can any operation increase total claimable value without a corresponding
   real asset entering the protocol?
   - Can totalSupply increase without mint() being explicitly called?
   - Can a user's balance increase without them depositing?
   - Are there rounding paths that create dust tokens over time?
   - Can anyone drain assets that were not theirs?

2. DOUBLE WITHDRAWAL
   Can the same value be withdrawn or redeemed more than once?
   - Is the user's claim decremented BEFORE or AFTER the external transfer?
   - Can withdraw() complete and then be called again successfully?
   - Is there any path where claim is not zeroed but funds are sent?

3. RESERVED FUNDS PROTECTION
   Are funds designated for specific purposes actually protected?
   - What funds are reserved? (escrow, locked deposits, etc.)
   - Can the owner or admin access these reserved funds through any function?
   - Does withdraw() or emergencyWithdraw() bypass the reservation check?

4. SOLVENCY
   Can total protocol liabilities ever exceed total protocol assets?
   - What does this protocol OWE to users? (list it)
   - What does this protocol HOLD as assets? (list it)
   - Under what conditions could OWED > HELD?

5. SHARE PRICE INTEGRITY
   Can the share/LP token price be artificially inflated?
   - Can a first depositor attack work? (deposit 1 wei, then donate large amount)
   - Can a donation directly to the contract change the share price?
   - Is the price formula stable across extreme reserve ratios?

6. ACCESS CONTROL
   Are all privileged operations properly gated?
   - List every function that changes critical state
   - For each: what is the exact access check? Where in the function?
   - Can any check be bypassed through proxy initialization, delegatecall,
     reentrancy during the check, or inheritance overrides?

7. STATE MACHINE CORRECTNESS
   Can operations happen in an invalid order?
   - What states does the contract have? (draw a list)
   - What transitions between states should be valid?
   - Can a function be called when the contract is in a state that
     makes the call invalid? (e.g., withdraw before deposit)

8. ACCOUNTING SYNCHRONIZATION
   Do all accounting systems stay synchronized?
   - Are there multiple variables tracking the same underlying value?
   - Can they become desynchronized through any sequence of operations?
   - What happens if an external call fails mid-accounting?

9. REWARD INTEGRITY
   Can rewards be claimed more than once?
   - How is "already claimed" tracked per user?
   - What happens if claimRewards() is called twice in the same block?
   - Is the claimable amount zeroed BEFORE the transfer executes?
   - Can a reentrancy attack during reward token transfer allow double-claim?

10. FUND LOCKUP PREVENTION
    Can funds ever become permanently unrecoverable?
    - If the contract is paused, can users withdraw through ANY path?
    - Is there an emergency withdrawal function? When is it callable?
    - Can governance or the owner lock the contract permanently?
    - What happens if a required external contract (oracle, token) fails?

FOR EACH CATEGORY PROVIDE:
- Vulnerability found? YES / NO
- If YES: exact function name and line, step-by-step exploit, financial impact ($)
- Plain-English property statement to add to design_doc.md
- Suggested fix (brief)
```

---

### Prompt B: Storage Layout and State Analysis
*Replaces AutoSetup Function #1: Solidity Compiler Analysis*

```
Analyze this Solidity contract for:

1. STORAGE LAYOUT
   List every state variable with:
   - Name and type
   - Storage slot (approximate)
   - Purpose in the protocol
   - Which functions read it / write it

2. FUNCTION ANALYSIS
   For every function list:
   - Visibility (public/external/internal/private)
   - Mutability (view/pure/payable/nonpayable)
   - External calls made (if any)
   - State variables modified
   - Reentrancy risk (low/medium/high and why)

3. REENTRANCY MAP
   For every external call:
   - Which function contains it?
   - Is state updated BEFORE or AFTER the external call?
   - Could an attacker exploit the callback window?

4. DEPENDENCY MAP
   For every external contract used:
   - Interface name and function called
   - What happens if this external call fails?
   - What happens if this external contract is malicious?

CONTRACT:
[PASTE CONTRACT]
```

---

### Prompt C: External Contract Classification
*Replaces AutoSetup Function #2: External Contract Classification*

```
For this Solidity project, classify all external contracts and imports:

[PASTE ALL CONTRACT FILES]

For each external or imported contract:
1. TYPE: ERC20 / ERC721 / Interface / Abstract / Library / Oracle / Other
2. IMPORTED BY: which contracts in the project use it
3. KEY FUNCTIONS CALLED: which functions from this external contract are called
4. POTENTIAL ISSUES:
   - Could this token have transfer fees?
   - Could this token be pausable?
   - Could this token have callbacks (ERC777, ERC1155)?
   - Is this a rebasing token?
5. CVL SUMMARY SUGGESTION: How should this be summarized in Certora?
   (DISPATCHER / CONSTANT / NONDET / custom summary)
```

---

### Prompt D: Harness Contract Generation
*Replaces AutoSetup Function #3: Harness Contract Generation*

```
Generate a CVL harness setup for testing [CONTRACT NAME].

Contract to test:
[PASTE CONTRACT]

External contracts it uses:
[LIST THEM]

Properties I want to test:
[LIST YOUR PROPERTIES FROM Step 4]

Please generate:
1. A harness contract (Solidity) that:
   - Inherits from the main contract
   - Adds ghost variables to track state across calls
   - Exposes helper functions for property testing

2. Mock implementations for each external dependency

3. A Certora .conf file with:
   - File paths and solc version
   - Method summaries for all external calls
   - verify directive pointing to main contract
   - Prover timeout set to 600 seconds

Use Solidity version 0.8.20.
```

---

## PART 7: PHASE B — AICOMPOSER PIPELINE (After AutoSetup)

### Setup (One Time After AutoSetup Access Granted)

```bash
# 1. Clone AutoSetup
git clone https://github.com/Certora/AutoSetup.git ~/AutoSetup

# 2. Set environment variable permanently
echo 'export AUTOSETUP_PATH=~/AutoSetup' >> ~/.bashrc
source ~/.bashrc

# 3. Verify
echo $AUTOSETUP_PATH
```

### Run the Pipeline

**PREREQUISITE:** `AUTOSETUP_PATH` must be set and AutoSetup cloned (see Setup section above).
The commands below **WILL FAIL** without AutoSetup configured.

```bash
# Standard audit run
tui-autoprove \
  ~/projects/protocol \
  src/Contract.sol:ContractName \
  design_doc.md \
  --cloud \
  --max-concurrent 4 \
  --cache-ns protocol-audit-v1

# Second run after updating design_doc (uses cache for unchanged parts)
tui-autoprove \
  ~/projects/protocol \
  src/Contract.sol:ContractName \
  design_doc.md \
  --cloud \
  --cache-ns protocol-audit-v1  # same namespace = reuses cache

# With more extraction rounds (default is 3, try 5 for complex protocols)
tui-autoprove \
  ~/projects/protocol \
  src/Contract.sol:ContractName \
  design_doc.md \
  --cloud \
  --max-bug-rounds 5 \
  --cache-ns protocol-audit-v1
```

### Phase Summary

| Phase | What Happens | Needs AutoSetup? |
|-------|-------------|-----------------|
| 0 | Reads code + design_doc, identifies components | NO |
| 1 | Harness generation, compiler analysis | YES — REQUIRED |
| 2 | CVL summaries for external contracts | YES (depends on Phase 1) |
| 3 | Structural invariants (total supply, etc.) | NO |
| 4 | Per-component property extraction (multi-round) | NO |
| 5 | CVL generation + Certora Prover verification | YES for harness |

### Output Files

```
certora/
├── specs/
│   ├── invariants.spec              # Structural invariants
│   └── autospec_{Component}.spec   # Per-component CVL specs
├── confs/
│   └── autospec_{Component}.conf   # Prover configuration
└── properties/
    ├── {Component}.properties.json      # Extracted properties
    ├── {Component}.property_rules.json  # Property → rule mapping
    └── {Component}.commentary.md        # LLM explanation of specs
```

---

## PART 8: COUNTER-EXAMPLE ANALYSIS

### Running the CEX Analyzer

```bash
cex-analyzer \
  certora/output/report_folder_name \
  ruleName \
  --method ContractName.functionName
```

### What cex_analyzer Shows

```
1. SUMMARY OF PROVING PROCESS
   What was checked and the exact path that led to the violation

2. DESCRIPTION OF SURFACED BUG
   Which property was violated and exactly why, with variable values

3. CONVERSATION ON HOW TO PROCEED
   Suggested fixes and alternative interpretations of the violation
```

### Prover Result Types

| Result | Meaning | Your Action |
|--------|---------|------------|
| VERIFIED | Property holds for all inputs | Record as confirmed safe |
| VIOLATED | Counter-example found — potential bug! (the Prover's actual status string is `VIOLATED`, not `FAILED`) | Investigate immediately with cex_analyzer |
| TIMEOUT | Too complex to finish | Simplify the rule, add more preconditions |
| SANITY_FAILED | The rule/invariant's assertion is unreachable — a precondition conflict, not a code bug | Check for vacuity; see `cex_instructions.j2` guidance |
| ERROR | The rule failed to run (e.g. a typecheck or setup error), not a property violation | Fix the underlying error and re-run |
| SKIPPED | The rule was not run this pass (e.g. filtered out, or blocked by an earlier error) | Check why it didn't run before treating it as passing |

### Translating Counter-Examples for Judges

Judges are often not formal verification experts. Use this format:

**Step 1: Plain-English Summary (No Jargon)**
```
The contract allows funds reserved for a specific purpose to be withdrawn
through the general withdrawal function, which means the reserved operation
can fail silently while funds have already been removed.
```

**Step 2: Concrete Scenario**
```
"Alice locks 100 tokens into the protocol's reserve. Before the reserved
operation executes, Alice calls withdraw(100 tokens). The contract allows
this because withdraw() does not check the reserved balance. The reserved
operation later fails — but the 100 tokens are already gone."
```

**Step 3: State Visualization**
```
Initial state:
  contract.balance   = 100 tokens
  reservedBalance    = 100 tokens
  availableBalance   = 0 tokens

Step 1: Alice calls withdraw(100 tokens)
  → withdraw() checks: 100 ≤ contract.balance ✓  (BUG: wrong check!)
  → Should check: 100 ≤ availableBalance          (which is 0)
  → Transfer executes: Alice receives 100 tokens

Final state:
  contract.balance   = 0 tokens    (WRONG — should still be 100)
  reservedBalance    = 100 tokens  (still says 100 is reserved — IMPOSSIBLE)
  availableBalance   = -100 tokens (INVALID STATE — cannot be negative)
```

**Step 4: Connect to Formal Verification**
```
This scenario was generated by Certora Prover as a counter-example
to the rule reservedFundsProtected(). The prover exhaustively verified
all possible inputs and found this as the minimal violation case.
This is not a theoretical edge case — it is the simplest possible
exploitation path.
```

---

## PART 9: REPORT WRITING — COMPLETE TEMPLATE

### The Multi-Layer Evidence Approach (From Our Conversation)

Every finding you submit should have THREE layers of evidence:
```
Layer 1: MATHEMATICAL PROOF (Certora counter-example)
         → Proves it's POSSIBLE (unrefutable)

Layer 2: DOCUMENTED CLAIM (from protocol's own documentation)
         → Proves it was PROMISED (makes invalidation impossible)

Layer 3: RUNNABLE PoC (Foundry test)
         → Proves it's EXPLOITABLE (shows real impact)
```

### Complete Finding Report Template

```markdown
# Finding: [Specific Title — What Can Happen, Not Just What's Wrong]

## Severity: [Critical / High / Medium / Low]

## Summary
[2–3 sentences in plain English. What is the vulnerability?
What can an attacker do because of it? Non-technical audience.]

## Protocol's Own Documentation
"[EXACT QUOTE from contract comment, NatSpec, or whitepaper]"
Source: [exact location]

This is the documented guarantee that is violated. The protocol
itself promised this behavior. The violation is not a matter of
interpretation.

## Vulnerability Details
[Technical explanation of WHY the guarantee is violated.
Which function, which check is missing, what path leads to violation.]

## Certora Formal Verification
The following CVL rule was generated by AIComposer and verified
by Certora Prover:

[PASTE THE CVL RULE from certora/specs/autospec_*.spec]

Prover result: FAILED
Counter-example: [paste or summarize the cex_analyzer output]

## How the Counter-Example Works
Step-by-step trace of the violation:

Initial state:
  [key variable] = [value]
  [key variable] = [value]

Step 1: [Actor] calls [function]([params])
  → [what happens]
  → [state change]

Step 2: [what changes next]

Final state:
  [key variable] = [value] ← INVALID
  [key variable] = [value]

[Explain what makes the final state invalid and dangerous]

## Impact
If exploited:
- Attacker can: [concrete action in plain English]
- Potential loss: [$ estimate based on current TVL or pool size]
- Affected users: [who is affected]
- Severity justification: [why this is Critical/High/Medium/Low
  per Immunefi/Sherlock standards]

## Proof of Concept
The following Foundry test demonstrates the vulnerability:

[PASTE YOUR FOUNDRY TEST — see Part 11 for format]

Run with:
forge test --match-test testVulnerabilityName -vvv

Expected output:
[Describe what the test shows when it passes]

## Addressing Potential Counter-Arguments

"Is this a false positive?"
The counter-example is realizable because:
- [Reason 1: e.g., no special permissions required]
- [Reason 2: e.g., standard ERC20 tokens involved]
- [Reason 3: e.g., publicly callable function]

"Is this intended behavior?"
The protocol's own documentation (quoted above) states the opposite.
This is a violation of the protocol's stated guarantees.

## Recommendation
[Specific code change to fix the issue. Reference the exact function.]

The fix ensures the property holds because:
[Explain why the fix resolves the violation]

## Additional Notes
Verified on: Certora Prover [version]
AIComposer version: [version]
Environment: All findings are verified counter-examples, not
theoretical violations.
```

---

## PART 10: FINDING CLASSIFICATION — 4 TYPES

### From Our Conversation — Critical for Report Framing

Before writing a report, classify your finding as one of these:

**Type 1: VULNERABILITY**
Direct exploitable issue with concrete financial impact.
```
Use when: You can show step-by-step how an attacker steals/freezes funds.
Label as: "This finding represents an exploitable vulnerability where..."
Severity: Critical or High
```

**Type 2: DESIGN FLAW**
Contract meets its formal specification but the design itself creates risk.
```
Use when: The code does what it says, but what it says is dangerous.
Label as: "While the contract meets its specification, this design pattern
           introduces risk because..."
Severity: Medium to High
```

**Type 3: SPECIFICATION VIOLATION**
Contract does not match its own documentation.
```
Use when: Code comment says X, but code actually does Y.
Label as: "The protocol documentation states [quote], but formal
           verification shows this constraint is not enforced."
Severity: High (documentation lies = trust broken)
```

**Type 4: CORRECTNESS ISSUE**
Logical inconsistency without a clear exploit path.
```
Use when: You found an invariant violation but can't make it pay.
Label as: "This invariant violation represents a logical inconsistency
           in the contract state. No direct exploit path was identified,
           however..."
Severity: Low to Medium
```

### Why Judges Invalidate Findings (and How to Prevent It)

| Reason for Invalidation | Prevention |
|------------------------|-----------|
| "No exploit path shown" | Always include a step-by-step scenario AND PoC test |
| "This is intended behavior" | Quote the protocol's OWN documentation that says otherwise |
| "False positive from prover" | Explain why counter-example is realizable in practice |
| "Insufficient impact" | Quantify dollar amount; cite Immunefi severity criteria |
| "No proof of concept" | Always include a running Foundry test |
| "Already known / out of scope" | Check existing audit reports before submitting |

---

## PART 11: POC TEST FORMAT — FOUNDRY

### Standard Foundry PoC Template

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/ContractUnderTest.sol";

contract VulnerabilityNamePoC is Test {
    ContractUnderTest internal protocol;
    address internal attacker = makeAddr("attacker");
    address internal admin = makeAddr("admin");

    function setUp() public {
        // Deploy contract
        protocol = new ContractUnderTest();

        // Initialize state
        vm.deal(admin, 100 ether);
        vm.startPrank(admin);
        protocol.initialize();
        vm.stopPrank();
    }

    function testVulnerabilityName() public {
        // ─── SETUP ───────────────────────────────────────────────
        vm.deal(attacker, 10 ether);
        vm.startPrank(attacker);

        // Record state BEFORE the exploit
        uint256 balanceBefore = address(protocol).balance;
        uint256 reservedBefore  = protocol.reservedBalance();

        console.log("=== BEFORE EXPLOIT ===");
        console.log("Contract balance:", balanceBefore);
        console.log("Reserved balance:", reservedBefore);
        console.log("Available:       ", protocol.availableBalance());

        // ─── EXPLOIT ──────────────────────────────────────────────
        // Step 1: [describe what attacker does first]
        protocol.deposit{value: 10 ether}();

        // Step 2: [describe the vulnerability trigger]
        protocol.withdraw(attacker, 10 ether);

        vm.stopPrank();

        // ─── VERIFY VIOLATION ─────────────────────────────────────
        uint256 balanceAfter = address(protocol).balance;
        uint256 reservedAfter  = protocol.reservedBalance();

        console.log("=== AFTER EXPLOIT ===");
        console.log("Contract balance:", balanceAfter);
        console.log("Reserved balance:", reservedAfter);
        console.log("Attacker gained: ", attacker.balance - 10 ether);

        // This assertion PASSES, showing the invariant IS violated
        assertLt(
            balanceAfter,
            reservedAfter,
            "VULNERABILITY: contract balance < reserved balance — invalid state"
        );
    }
}
```

**Run command:**
```bash
forge test --match-test testVulnerabilityName -vvv
```

### Copilot Pro PoC Writing Prompt

```
I found a potential vulnerability in this smart contract:

DOCUMENTED CLAIM: [exact quote from contract or whitepaper]
PROPERTY VIOLATED: [plain English statement of what should be true]
HOW IT'S VIOLATED: [which function, what check is missing]
COUNTER-EXAMPLE: [paste cex_analyzer output or your trace]

Write a complete Foundry test (PoC) that demonstrates this vulnerability.

Requirements:
- Use vm.startPrank() for actor impersonation
- Record state with console.log() BEFORE and AFTER the exploit
- Use descriptive variable names (balanceBefore not b1)
- The test should PASS to prove the bug exists (not revert)
- The final assertion should show the invariant IS violated
- Include a comment on each step explaining what the attacker does
- Include a comment explaining what SHOULD have happened instead

Also provide the one-line fix that would make this test FAIL
(i.e., the fix that closes the vulnerability).
```

---

## PART 12: CONTEST SUBMISSION STRATEGY

### The Key Insight (From Our Conversation)

```
When you say:
  "This is a bug"
Judges can say:
  "Intended behavior" → INVALIDATED

When you say:
  "The protocol's own documentation at [source] states [claim].
   Certora Prover formally proves this claim is violated.
   Here is the mathematical counter-example.
   Here is the runnable test."
Judges cannot say:
  "Intended behavior" → MUST BE VALIDATED
```

### Packaging Your Submission

Every submission should include:

```
submission/
├── AUDIT_REPORT.md          # Your full report (Part 9 template)
├── design_doc.md            # Shows your research methodology
├── certora/
│   └── specs/
│       └── autospec_*.spec  # Generated CVL specs (proof)
├── test/
│   └── Exploit.t.sol        # Runnable PoC test
└── README.md                # How to run the PoC
```

### README.md for Your Submission

```markdown
# Bug Bounty Submission — [Protocol Name]

## How to Reproduce

### Prerequisites
- Foundry installed
- Protocol repository cloned

### Run the PoC
git clone [protocol repo]
cd [protocol]
forge install
forge test --match-test testVulnerabilityName -vvv

### Expected Output
[Describe what the test outputs that proves the bug]

### How Formal Verification Found This
This vulnerability was identified through formal verification using
Certora Prover via AIComposer. The CVL rule [rule_name] in
certora/specs/autospec_[component].spec produces a counter-example
that corresponds to the PoC test above.
```

---

## PART 13: CATASTROPHIC FAILURE SCENARIOS TEMPLATE

Add this section to EVERY design_doc.md you write. It forces you to think
about the five worst possible outcomes and what prevents each one. AIComposer
uses these to generate its highest-priority verification targets.

> Authoritative version: BUSINESS_LOGIC_CORE.md Part 13.

```markdown
## Catastrophic Failure Scenarios

### CF-1: Theft of Reserved Funds
Impact: Loss of reserved protocol funds. Core operations permanently broken.
Estimated loss: [$ amount based on TVL]

Must Never Happen:
Any withdrawal or transfer reduces the reserved balance.
Any actor accesses reserved funds through non-intended paths.

Affected Functions:
withdraw(), emergencyWithdraw(), transfer(), [any function that moves funds]

What AIComposer should verify:
After any call to withdraw(), reservedBalance is unchanged.
Reserved funds can only be reduced through their intended mechanism.

---

### CF-2: Protocol Insolvency
Impact: User claims exceed protocol assets. Protocol cannot honor withdrawals.
Estimated loss: [$ amount — potentially entire TVL]

Must Never Happen:
Total liabilities (what the protocol owes) exceeds total assets (what it holds).

Affected Functions:
deposit(), withdraw(), borrow(), repay(), liquidate(), getHealthFactor()

What AIComposer should verify:
At all times: sum(user_claims) ≤ address(this).balance + external_assets.
No sequence of valid operations makes the protocol insolvent.

---

### CF-3: Share / Accounting Inflation
Impact: Attacker obtains disproportionate ownership of the protocol.
Estimated loss: [$ amount per attack]

Must Never Happen:
Shares minted to a user exceed the value of assets they deposited.
First depositor can set an initial share price that harms others.

Affected Functions:
deposit(), mint(), redeem(), withdraw(), convertToShares(), convertToAssets()

What AIComposer should verify:
Share price remains stable before and after any deposit.
Shares minted always proportional to assets received.

---

### CF-4: Permanent Lockup
Impact: User funds become permanently unrecoverable.
Estimated loss: [$ amount — all user deposits]

Must Never Happen:
A state exists where withdraw() fails AND emergencyWithdraw() fails.
Owner can freeze the contract with no recovery mechanism.

Affected Functions:
withdraw(), emergencyWithdraw(), pause(), unpause()

What AIComposer should verify:
For any user with balance > 0, at least one withdrawal path succeeds.
Pause does not block ALL fund recovery paths simultaneously.

---

### CF-5: Unauthorized Control
Impact: Attacker gains privileged authority over the protocol.
Estimated loss: [$ amount — entire protocol TVL at risk]

Must Never Happen:
Non-owner successfully calls any owner-only function.
Ownership transfers to an unauthorized address.

Affected Functions:
pause(), unpause(), setFees(), addAuthorizedDepositor(),
transferOwnership(), upgradeTo()

What AIComposer should verify:
Every privileged function reverts for non-authorized callers.
Ownership cannot change without explicit action from current owner.
```

---

## PART 14: COMPLETE WORKED EXAMPLE

This is a complete worked example showing how to apply all steps to a
real smart contract. The pattern applies to any protocol — DeFi, staking,
lending, governance, or NFT.

### Step 2 Output: Claims Extracted from the Contract

```
CLAIM 1: "Designed to handle deposits with a designated operator"
SOURCE:  Contract description comment
TYPE:    Access control
→ MEANING: Only designated entities can trigger critical operations

CLAIM 2: "A portion of funds is locked and cannot be withdrawn"
SOURCE:  Comment in the contract
TYPE:    Fund protection
→ MEANING: Locked/reserved funds are unavailable for general withdrawal

CLAIM 3: "Available balance excludes reserved amounts"
SOURCE:  @notice comment on availableBalance()
TYPE:    Accounting
→ MEANING: availableBalance = contract.balance − reservedBalance

CLAIM 4: Implicit from onlyOwner modifier on withdraw()
TYPE:    Access control
→ MEANING: Only owner can withdraw available funds

CLAIM 5: "Pauses critical operations"
SOURCE:  Function description for pause()
TYPE:    State transition
→ MEANING: When paused, critical operations must revert

CLAIM 6: "Locked funds are set aside for specific use later" +
         "Unlock returns funds back to available balance"
SOURCE:  Function descriptions for lock() and unlock()
TYPE:    Accounting
→ MEANING: lock() increases reservedBalance, unlock() decreases it

CLAIM 7: "The contract becomes immutable after finalization"
SOURCE:  Context from finalize() function
TYPE:    State machine
→ MEANING: finalize() is a permanent, irreversible state change
```

### Step 4 Output: Formal Properties

```
PROPERTY 1: Balance Invariant
The contract's total balance must always equal the sum of the
available balance plus the reserved balance. After every function call,
this equality must hold. If it does not, the contract's accounting
has become desynchronized — there are funds that are neither
available nor properly reserved.

PROPERTY 2: Withdrawal Access Control
The withdraw() function can only be called by the owner address.
Any caller other than the owner must cause the transaction to revert.
There is no path through any other function that allows a non-owner
to withdraw funds from the contract.

PROPERTY 3: Reserved Balance Cannot Be Withdrawn
When the owner calls withdraw(), the reserved balance must not decrease.
The amount withdrawn must not exceed the available balance, where
available balance is defined as the contract's total balance
minus the reserved balance. After withdraw() completes, the reserved
balance must be exactly the same as it was before the call.

PROPERTY 4: Pause Enforcement
When the paused flag is true, calls to critical operations must
always revert. No state-changing operations should succeed while
the contract is paused.

PROPERTY 5: Lock Increases Reserved Balance
A successful call to lock(amount) increases reservedBalance by exactly
amount and decreases availableBalance by exactly amount.
Total contract balance does not change.

PROPERTY 6: Unlock Decreases Reserved Balance
A successful call to unlock(amount) decreases reservedBalance by exactly
amount and increases availableBalance by exactly amount. The total
contract balance does not change, since unlock() only reclassifies
funds from reserved to available — no funds enter or leave the contract.

PROPERTY 7: Operator-Only Access to Critical Functions
Only the authorized operator address can call lock(), unlock(),
and executeOperation(). All other callers must revert.
There is no path through any other function that allows a non-operator
to reserve funds or trigger a critical operation.

PROPERTY 8: Available Balance Cannot Be Negative
The available balance calculated as the contract's total balance minus
the reserved balance must never be negative. This would indicate that
reservedBalance was incremented beyond the actual funds held, which would
mean the contract's accounting has lost synchronization. A negative
available balance would make it impossible to determine how much
can safely be withdrawn without touching reserved funds.
```

### Step 5 Output: Pre/Postconditions

#### withdraw(address recipient, uint256 amount)

```
PRECONDITIONS:
- msg.sender == owner
- recipient != address(0)
- amount > 0
- amount ≤ availableBalance()
- availableBalance() == address(this).balance − reservedBalance

POSTCONDITIONS:
- recipient.balance increased by exactly amount
- address(this).balance decreased by exactly amount
- reservedBalance is UNCHANGED (critical — reserved funds are protected)
- Withdrawal event is emitted

REVERTS IF:
- msg.sender != owner (onlyOwner modifier)
- recipient == address(0)
- amount == 0
- amount > availableBalance()
- Transfer to recipient fails
```

#### lock(uint256 amount)

```
PRECONDITIONS:
- msg.sender == operator
- paused == false
- amount > 0
- amount ≤ availableBalance()

POSTCONDITIONS:
- reservedBalance increased by exactly amount
- availableBalance decreased by exactly amount
- address(this).balance UNCHANGED (funds stay in contract)
- FundsLocked event is emitted

REVERTS IF:
- msg.sender != operator
- paused == true
- amount == 0
- amount > availableBalance()
```

#### executeOperation(Operation calldata op)

```
PRECONDITIONS:
- msg.sender == operator
- paused == false
- op.amount > 0
- op.amount ≤ availableBalance()
- TARGET_CONTRACT address is set and valid

POSTCONDITIONS:
- Operation submitted to target contract
- address(this).balance decreased by op.amount
- OperationExecuted event emitted

REVERTS IF:
- msg.sender != operator
- paused == true
- op.amount > availableBalance()
- TARGET_CONTRACT call fails
```

---

## PART 15: WSL UBUNTU SETUP REFERENCE

### Environment Variables (Add to ~/.bashrc)

```bash
export ANTHROPIC_API_KEY="sk-ant-..."     # Required: AIComposer won't start without this
export CERTORAKEY="..."                    # Required for cloud prover mode
export AUTOSETUP_PATH="~/AutoSetup"       # Required: set after Certora grants access
export CERTORA_AI_COMPOSER_PGHOST="localhost"
export CERTORA_AI_COMPOSER_PGPORT="5432"
```

### One-Time Setup Commands

```bash
# 1. Install dependencies
sudo apt update
sudo apt install -y python3.12 python3.12-venv python3-pip git

# 2. Install uv
pip install uv

# 3. Clone AIComposer
cd ~
git clone https://github.com/Certora/AIComposer.git
cd AIComposer

# 4. Start PostgreSQL (Docker Desktop must be running on Windows
#    with WSL integration enabled in Docker Desktop settings)
cd scripts/
docker compose create && docker compose start

# 5. Build RAG database (takes ~15 minutes, run once)
./gen_docs.sh
./populate_rag.sh

# 6. Install Solidity compilers
pip install solc-select
solc-select install 0.8.29
ln -s ~/.solc-select/artifacts/solc-0.8.29/solc /usr/local/bin/solc8.29

# 7. Install AIComposer
cd ~/AIComposer
uv sync --group ml

# 8. Verify
python -c "import composer; print('AIComposer ready')"
```

### Docker Desktop WSL Integration (Critical)

```
Open Docker Desktop on Windows
→ Settings → Resources → WSL Integration
→ Enable integration with your Ubuntu distribution
→ Apply & Restart

Then in WSL terminal:
docker ps    # Should work without sudo
```

---

## PART 16: QUICK REFERENCE CHECKLIST

### Before Starting Any Audit

- [ ] Target protocol selected (has public docs/whitepaper)
- [ ] VS Code open with contract file and Copilot Pro active
- [ ] Notepad/file ready for extracted claims (Step 2)

### During Property Extraction (7 Steps)

- [ ] Step 1: Copilot summarized contract (understand before extracting)
- [ ] Step 2: YOU manually extracted claims from comments/require/modifiers
- [ ] Step 3: Copilot mapped 3–6 components
- [ ] Step 4: YOU converted each claim to a formal property (plain English)
- [ ] Step 5: Pre/postconditions defined for 3+ critical functions
- [ ] Step 6: design_doc.md drafted — ZERO code in it
- [ ] Step 7: Copilot reviewed doc — all 10 categories addressed

### Before Running AIComposer

- [ ] design_doc.md is 2000+ words
- [ ] design_doc.md has NO CVL or Solidity code
- [ ] All 10 business logic categories have entries
- [ ] Edge cases listed (zero, max, empty, paused states)
- [ ] ANTHROPIC_API_KEY set in WSL
- [ ] CERTORAKEY set in WSL
- [ ] AUTOSETUP_PATH set in WSL (after access granted)
- [ ] Docker PostgreSQL running: `docker ps | grep pgvector`
- [ ] RAG database populated (one-time setup done)

### After Getting Prover Results

- [ ] All FAILED rules investigated with cex_analyzer
- [ ] Each failure verified as real (not a proof artifact)
- [ ] Finding classified as Vulnerability / Design Flaw / Spec Violation / Correctness
- [ ] PoC test written and passing (using Part 11 template)
- [ ] Impact quantified in dollars
- [ ] Severity justified using Immunefi/Sherlock criteria
- [ ] Report written using Part 9 template
- [ ] Finding ties back to documented protocol claim
- [ ] Submission package prepared (report + specs + PoC + README)

---

*Master Audit Guide V2 — Corrected and Comprehensive*
*Combines all 18 items from conversation + 5 valuable items from uploaded docs*
*16 Parts covering the complete end-to-end workflow*