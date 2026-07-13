# Certora AIComposer — Getting Started

> **Single entry point** for new and returning users. Three levels of depth — start at Level 1 and go deeper as needed.
>
> **Key insight**: The highest-value findings in modern audits ($10k-$50k+) come from **business logic bugs** — broken invariants around asset conservation, solvency, share price integrity, and state machine correctness — not reentrancy or overflow bugs that static analyzers already catch. See [USER_GUIDE.md](./USER_GUIDE.md) for the full 10-category framework.

---

## Level 1: Run Your First Audit (5 minutes)

### The One Command

```bash
console-autoprove . src/Contract.sol:ContractName design_doc.md --cloud
```

That's it. This will:
1. Read your design document
2. Extract 10-20 security & business logic properties
3. Generate CVL specifications
4. Run the Certora Prover
5. Show you VERIFIED / FAILED / TIMEOUT / SKIPPED results

**Time**: 5-15 minutes. **Result**: Properties extracted, CVL generated, tests run.

### Prerequisite Check (90 seconds)

```bash
python3 --version                        # Need 3.12+
echo $ANTHROPIC_API_KEY                 # Must be set
echo $CERTORAKEY                         # For cloud mode
docker ps | grep postgres               # DB should be running
```

If anything fails, see [README.md](./README.md) for full installation.

### What the Results Mean

| Result | Meaning | Action |
|--------|---------|--------|
| ✅ **VERIFIED** | Property holds, no bug | OK |
| ❌ **FAILED** | Counterexample found! | **INVESTIGATE!** |
| ⏱ **TIMEOUT** | Too complex to verify | Simplify property |
| ⊘ **SKIPPED** | Not formalizable | May indicate false negative |

---

## Level 2: Understand the Pipeline (15 minutes)

### System Architecture

```
Input: Design Document + Solidity Contract
                    │
    ┌───────────────┼───────────────┐
    │               │               │
    ▼               ▼               ▼
Phase 0-1:      Phase 2-3:      Phase 4-5:
System          Setup &         Properties
Analysis        Summaries       & CVL Gen
    │               │               │
    └───────────────┼───────────────┘
                    ▼
            Prover Verification
            • VERIFIED ✓
            • FAILED ✗ (potential bugs!)
            • TIMEOUT ⏱
            • SKIPPED ⊘
```

### The Two Workflows

AIComposer has **two complementary subsystems**:

| System | Direction | Input | Output |
|--------|-----------|-------|--------|
| **Auto-Prove** | Code → Specs | Source code + design doc | CVL specs + prover results |
| **AIComposer (Codegen)** | Specs → Code | CVL spec + interface + system doc | Solidity implementation |

**Auto-Prove** is what you use for bug hunting. **Codegen** is for synthesizing verified implementations.

### Auto-Prove Pipeline Phases

```
Phase 0: System Analysis — LLM identifies components, contracts, external actors
Phase 1: Harness Setup — AutoSetup classifies external contracts, generates config
Phase 2: Custom Summaries — CVL summaries for ERC20s and external interfaces
Phase 3: Structural Invariants — System-wide invariants (e.g., total supply)
Phase 4: Property Extraction — Per-component, multi-round property discovery
Phase 5: CVL Generation — Formalize properties, judge review, prover feedback loop
```

### Property Types

| Type | Example | CVL |
|------|---------|-----|
| **Invariant** | "Sum of balances = total supply" | `invariant` keyword, always true |
| **Safety Property** | "Deposit increases balance" | `rule` keyword, expected behavior |
| **Attack Vector** | "Oracle can't manipulate price >5%" | `rule` with adversarial assumptions |

### Multi-Round Convergence

```
Round 1: Broad sweep → 10-15 properties
Round 2: New angles → 3-5 new properties (no duplication)
Round 3: Convergence → Empty output = done!
```

Empty Round 3 is **good** — it means you've exhaustively found all properties for that component.

---

## Level 3: Write Effective Design Documents (30 minutes)

### Why This Matters (The 80% Rule)

**Design document quality is 80% of success.** A vague doc produces generic properties and misses real bugs. A specific doc produces targeted properties that find real vulnerabilities.

```
EXCELLENT (5000+ words, detailed edges)  → 70-80% bug discovery rate
GOOD (2000-3000 words, specifics)        → 40-50% bug discovery rate
POOR (500-1000 words, generic)           → 10-20% bug discovery rate
TERRIBLE (vague, incomplete)             → 0-5% useful findings
```

### The Auditor Mindset

When writing your design document, don't just describe what the contract does. **Hunt for impact.** For every guarantee, ask:

> *"If this invariant breaks, can someone **steal money, print money, freeze money, or brick the protocol**?"*

These answers become your highest-value properties. The most severe modern DeFi exploits come from broken business logic invariants — not reentrancy or overflow bugs that static analyzers already catch.

### Design Document Template

```markdown
# [Protocol Name] Design Document

## 1. System Overview
[2-3 sentences: What is this protocol? What does it do?]

## 2. Components
### Component: [Name]
- Purpose: [What does it do?]
- Functions: [list key functions]
- State: [list state variables]
- External Dependencies: [oracle, tokens, etc.]
- Risk Areas: [What could go wrong?]
[Repeat for each component]

## 3. Stated Guarantees (with Impact)
For each guarantee, state what happens if it breaks:

"LP tokens are always redeemable for proportional share"
  → IMPACT IF VIOLATED: Share dilution → depositor funds stolen (CRITICAL)

"Fees are collected correctly and never lost"
  → IMPACT IF VIOLATED: Fee theft → protocol revenue loss (CRITICAL)

"Only owner can pause the protocol"
  → IMPACT IF VIOLATED: Unauthorized pause → funds frozen (HIGH)

## 4. Properties (Claim → Property Format)
For each business logic category, state the claim, then the property:

### Asset Conservation
CLAIM: Assets cannot be created from nothing.
PROPERTY: Total assets accounted for must never exceed actual assets held.
PROPERTY: A user cannot increase their claim without providing equivalent value.

### Solvency
CLAIM: Protocol remains fully collateralized.
PROPERTY: Total liabilities must never exceed total assets.
PROPERTY: A user cannot borrow more than allowed collateral.

### Share Price Integrity
CLAIM: Share value cannot be artificially inflated.
PROPERTY: External transfers must not artificially inflate share value.
PROPERTY: Share issuance must be proportional to assets deposited.

[Continue for all 10 categories — see USER_GUIDE.md for the full list]

## 5. Edge Cases & State Transitions
- Boundary conditions: What if reserve is 0? What if deposit is 1 wei?
- State transitions: Pause→Resume safety, Emergency→Normal consistency
- Reentrancy points: Where are external calls?
- Oracle manipulation: Staleness, zero price, extreme values

## 6. Catastrophic Failure Scenarios

### CF-1: Theft of Reserved Funds
Impact: Loss of beacon-chain deposit funds. CRITICAL.
Must Never Happen: Owner withdrawal uses staged ETH.
Affected: withdraw(), stage(), depositToBeaconChain()

### CF-2: Insolvency
Impact: User claims exceed protocol assets. CRITICAL.
Must Never Happen: Total liabilities > total assets.
Affected: borrow(), deposit(), withdraw(), liquidate()

### CF-3: Share Inflation
Impact: Attacker obtains disproportionate ownership. CRITICAL.
Must Never Happen: Shares minted exceed value deposited.
Affected: deposit(), mint(), convertToShares()

### CF-4: Permanent Lockup
Impact: Funds become unrecoverable. HIGH.
Must Never Happen: All withdrawal paths become inaccessible.
Affected: withdraw(), emergencyWithdraw(), pause()

### CF-5: Unauthorized Control
Impact: Attacker gains privileged authority. CRITICAL.
Must Never Happen: Non-owner performs owner-only action.
Affected: pause(), setFees(), upgradeTo()

## 7. Assumptions & Limitations
- What's out of scope
- What we assume is safe
```

### Design Doc Quality Checklist

Before running AIComposer, verify:
- [ ] 2000+ words minimum
- [ ] Each component has purpose, functions, state, risks
- [ ] Every guarantee states its IMPACT IF VIOLATED
- [ ] Properties written in Claim→Property format
- [ ] At minimum: asset conservation, solvency, share price integrity, state machine, and access control categories addressed
- [ ] Edge cases explicitly flagged
- [ ] State transitions considered
- [ ] Catastrophic Failure Scenarios use "Must Never Happen" + "Affected" format
- [ ] Assumptions listed

---

## 4 Common Scenarios

### Scenario 1: Quick Bug Hunt
```bash
console-autoprove . src/C.sol:C design_doc.md --cloud
```

### Scenario 2: With Threat Model (Focus Attack Surface)
```bash
console-autoprove . src/C.sol:C design_doc.md --threat-model threat.md --cloud
```

### Scenario 3: Interactive Refinement
```bash
console-autoprove . src/C.sol:C design_doc.md --interactive --cloud
```

### Scenario 4: Multi-Pass (Multiple Attack Angles)
```bash
# Pass 1: Oracle attacks
console-autoprove . src/C.sol:C design_doc.md --threat-model oracle_threat.md --cache-ns oracle --cloud
# Pass 2: Access control
console-autoprove . src/C.sol:C design_doc.md --threat-model access_threat.md --cache-ns access --cloud
```

---

## Bug Hunt Workflow (Step by Step)

```
1. RESEARCH (1-2 hours)
   → Read whitepaper, understand protocol, identify risky areas

2. DESIGN DOC (1-2 hours)
   → Copy template above, fill in details, flag edge cases

3. RUN EXTRACTION (5-15 minutes)
   → console-autoprove . src/C.sol:C design_doc.md --cloud

4. ANALYZE RESULTS (30-60 minutes)
   → Look for FAILED rules, read counterexamples, check if realistic

5. VERIFY FINDING (30-60 minutes)
   → Check actual code, confirm exploit works, write clear report

6. SUBMIT (30 minutes)
   → Clear steps to reproduce, impact assessment, recommended fix

TOTAL: 4-6 hours per protocol
```

---

## Common Errors & Fixes

| Error | Fix |
|-------|-----|
| `AUTOSETUP_PATH not set` | Add `--cloud` flag |
| `ANTHROPIC_API_KEY not set` | `export ANTHROPIC_API_KEY=sk-...` |
| `PostgreSQL connection failed` | `cd scripts && docker compose start` |
| `No solc compiler found` | Install solc with naming convention `solcX.Y` (e.g., `solc8.29`) |
| `CERTORAKEY not set` | Set for cloud mode: `export CERTORAKEY=...` |

---

## Working Without AutoSetup

AutoSetup (private Certora infrastructure) handles compilation analysis and harness generation. **You don't need it.** Use `--cloud` for cloud prover (no setup, 10x faster), or use the manual workflow:

1. Analyze contract with Claude for storage layout, external dependencies, and business logic bugs
2. Generate CVL harness and `.conf` file with Claude
3. Extract properties via `console-autoprove ... --cloud`
4. Combine security + business logic properties into comprehensive CVL
5. Run prover

See [USER_GUIDE.md](./USER_GUIDE.md) for the complete manual workflow with all 10 business logic categories.

---

## The 10 High-Value Business Logic Bug Categories

Based on research of $1B+ real-world DeFi exploits. These cause ~80% of major losses.

| # | Category | Example | Severity |
|---|----------|---------|----------|
| 1 | **Asset Conservation** | Inflation/double-mint | CRITICAL |
| 2 | **Double Withdrawal** | Same value redeemed twice | CRITICAL |
| 3 | **Reserved Funds** | Beacon ETH protected from access | CRITICAL |
| 4 | **Solvency** | Total liabilities ≤ total assets | CRITICAL |
| 5 | **Share Price Integrity** | Cannot inflate share value | CRITICAL |
| 6 | **Access Control** | Only authorized actors | CRITICAL |
| 7 | **State Machine** | Valid transitions only | HIGH |
| 8 | **Accounting Sync** | Multi-contract consistency | CRITICAL |
| 9 | **Reward Integrity** | Cannot double-claim rewards | HIGH |
| 10 | **Fund Lockup Prevention** | Recovery always possible | HIGH |

Full details with CVL examples: [USER_GUIDE.md](./USER_GUIDE.md)

---

## Severity Framework (Immunefi/Sherlock/HackenProof Standards)

| Severity | Bounty | Requirements |
|----------|--------|-------------|
| **CRITICAL** | $10k-$50k+ | Direct theft, permanent freezing, insolvency, unauthorized minting |
| **HIGH** | $5k-$20k | Temporary freezing >1 month, unclaimed yield theft, broken accounting |
| **MEDIUM** | $1k-$5k | DoS >1 week, state inconsistency, broken returns |
| **LOW** | <$1k | Gas issues, informational |

---

## Pro Tips

💡 **Cache for Iteration**: Update design doc, re-run with same `--cache-ns`. Cache auto-invalidates when doc changes.

💡 **Multi-Pass Coverage**: Run with different threat models to cover more attack surface.

💡 **Simplify If Timeout**: Reduce concurrent agents: `--max-concurrent 2`.

💡 **Verify Findings**: Don't trust prover alone — check code + runtime.

💡 **Design doc is king**: The single most important factor in finding real bugs.

---

## Next Steps: Where to Go From Here

| You want to... | Read |
|---|---|
| **Start auditing TODAY** (step-by-step manual workflow) | **[AUDIT_WORKFLOW.md](./AUDIT_WORKFLOW.md)** ← START HERE |
| Full master reference (Copilot prompts, templates) | [MASTER_AUDIT_GUIDE_V2.md](./MASTER_AUDIT_GUIDE_V2.md) |
| The 10 business logic categories | [BUSINESS_LOGIC_CORE.md](./BUSINESS_LOGIC_CORE.md) |
| All commands, flags, and error codes | [REFERENCE.md](./REFERENCE.md) |
| Real vulnerability case studies | [REAL_VULNERABILITY_EXAMPLES.md](./REAL_VULNERABILITY_EXAMPLES.md) |
| Install/setup AIComposer | [README.md](./README.md) |
| Understand pipeline internals | [AUTOPROVE.md](./AUTOPROVE.md) |

---

## TL;DR — Ultra Condensed

```
1. Write design_doc.md (2000+ words, specific, with catastrophic failure scenarios)
2. Run: console-autoprove . src/C.sol:C design_doc.md --cloud
3. Look for FAILED rules
4. Investigate failures in code
5. Verify exploit works
6. Submit findings
```
