# Certora AI Composer

AI Composer is a tool for generating verified implementations from documentation and CVL specifications, with advanced CEX (Counterexample) analysis powered by Claude AI.

---

## 🎯 Quick Navigation

### **New Users → Start Here**
- **Want to extract and analyze counterexamples?** → See [CEX Extraction & Analysis Tool](#-cex-extraction--analysis-tool)
- **Want to generate verified code?** → See [Full Installation](#full-installation--aicomposer)
- **Want setup automation?** → See [Quick Start](#-quick-start-3-minutes)

### **Returning Users**
- [CEX Tool Commands](#usage-commands)
- [Documentation](#documentation)
- [Troubleshooting](#troubleshooting)

---

## 🚀 CEX Extraction & Analysis Tool

### What Is It?

A production-grade tool that automatically:
1. **Extracts** counterexamples from Certora Prover JSON output
2. **Analyzes** them using Claude AI
3. **Suggests** code fixes

**Used by:** DeFi developers, formal verification engineers, smart contract auditors

**Benefit:** Turn hours of manual CEX analysis into 30 seconds of automation

---

## ⚡ Quick Start (3 Minutes)

### Prerequisites
- Python 3.9+
- `pip` 
- Anthropic API key (get one from [console.anthropic.com](https://console.anthropic.com/account/keys))

### Step 1: Download Setup Script

```bash
curl -o setup_certora_tools.sh \
  https://raw.githubusercontent.com/0xbrett8571/CertoraAIComposer/master/setup_certora_tools.sh

chmod +x setup_certora_tools.sh
```

### Step 2: Run Setup (One Time Only)

```bash
./setup_certora_tools.sh
```

The script will:
- ✅ Create isolated Python environment
- ✅ Install dependencies
- ✅ Download the CEX tool
- ✅ Configure your API key
- ✅ Set up shell aliases
- ✅ Verify everything works

When prompted, enter your Anthropic API key: `sk-ant-...`

### Step 3: Reload Shell

```bash
source ~/.bashrc  # or ~/.zshrc
```

### Step 4: Verify Setup

```bash
certora-verify
```

Expected output:
```
Verifying Certora setup...
✓ Tools Home: /Users/yourname/certora-tools
✓ Python Env: /Users/yourname/certora-tools/venv
✓ CEX Tool: /Users/yourname/certora-tools/extract_and_analyze_cex.py
✓ API Key: sk-ant-XXXXXXXX...
✓ Model: claude-3-5-sonnet-20241022
```

---

## 📖 Usage Commands

### Command 1: List All Rules

See which rules passed/failed verification:

```bash
cex-list /path/to/prover_results
```

Output:
```
Available Rules in Treeview:

Rule Name                                Status          CEX Available  
----------------------------------------------------------------------
transfer_succeeds                        VIOLATED        ✓ Yes          
balanceInvariant                         VERIFIED        ✗ No           
noNegativeBalance                        VIOLATED        ✓ Yes          
reentrancyGuard                          TIMEOUT         ✗ No           
```

**When to use:** Right after running Certora to see what failed

---

### Command 2: Extract Counterexample

Get detailed CEX trace for a specific failed rule:

```bash
cex-extract /path/to/prover_results transfer_succeeds
```

Output:
```
======================================================================
COUNTEREXAMPLE FOR RULE: transfer_succeeds
======================================================================

→ Calling function Bank.transfer(to, 1000)
  → Entering branch: balances[msg.sender] < 1000
    → Assert failed: balances[to] == balances_before[to] + 1000
      → Context: balances[msg.sender] = 500, amount = 1000

======================================================================
```

**When to use:** When you need to understand what inputs trigger a failure

---

### Command 3: Analyze with Claude (The Magic!)

Get Claude's analysis of why a rule failed and how to fix it:

```bash
cex-analyze /path/to/prover_results transfer_succeeds \
  --spec ./spec/Token.spec \
  --output analysis.md
```

Output saved to `analysis.md`:
```
======================================================================
CLAUDE ANALYSIS FOR RULE: transfer_succeeds
======================================================================

## Scenario
The counterexample shows a situation where a user attempts to transfer
1000 tokens when their balance is only 500 tokens. The rule expects the
transfer to succeed and update the recipient's balance.

## Root Cause
Line 45 in Token.sol performs:
    balances[to] += amount;
    
Without first checking:
    require(balances[msg.sender] >= amount, "Insufficient balance");

This allows the sender's balance to go negative.

## Potential Fixes

### Fix 1: Add balance check (Recommended)
    function transfer(address to, uint amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }

### Fix 2: Use SafeMath library
    balances[msg.sender] = balances[msg.sender].sub(amount);

### Fix 3: Implement OpenZeppelin's ERC20Transfer
    _transfer(msg.sender, to, amount);

## Confidence
HIGH - The root cause is straightforward: missing balance validation

======================================================================
```

**When to use:** To get specific code fix suggestions from Claude

---

## 🔄 Typical Workflow (Day-by-Day)

### Morning: Run Prover

```bash
cd ~/my-defi-protocol
certoractl run config.conf -o ./prover_results
# ⏳ Wait 5 minutes to 2 hours (grab coffee ☕)
```

### Mid-Morning: See What Failed

```bash
cex-list ./prover_results
# Shows: 3 rules VIOLATED, 2 rules VERIFIED
```

### Late Morning: Analyze First Failure

```bash
cex-analyze ./prover_results transfer_succeeds \
  --spec ./spec/Token.spec \
  --output analysis.md

cat analysis.md  # Read the analysis
```

### Noon: Apply Fix

```bash
vim contracts/Token.sol
# Add require statement based on Claude's suggestion
```

### Afternoon: Re-verify

```bash
certoractl run config.conf -o ./prover_results_v2

cex-list ./prover_results_v2
# Check: is transfer_succeeds now VERIFIED?
```

### End of Day: Repeat for Next Failure

```bash
cex-analyze ./prover_results_v2 noNegativeBalance \
  --spec ./spec/Token.spec \
  --output analysis_v2.md
```

---

## 📊 Batch Processing (Analyze All Failures at Once)

Analyze all failed rules automatically:

```bash
#!/bin/bash
# Save as scripts/analyze_all.sh

cex-list ./prover_results | grep VIOLATED | awk '{print $1}' | while read rule; do
    cex-analyze ./prover_results "$rule" \
      --spec ./spec/*.spec \
      --output "analysis_${rule}.md"
    echo "✓ Analyzed: $rule"
done

echo "✅ All analyses saved!"
ls -lh analysis_*.md
```

Run it:
```bash
chmod +x scripts/analyze_all.sh
./scripts/analyze_all.sh
```

---

## 📁 Per-Project Setup

Each project needs minimal setup (2 minutes):

```bash
cd ~/my-new-defi-project

# Create project marker file
cat > .certora-project << 'EOF'
PROJECT_NAME="my-new-defi-project"
CERTORA_OUTPUT_DIR="./prover_results"
SPEC_DIR="./spec"
MAIN_CONTRACT="MyContract"
EOF

# That's it! Now you can use all commands in this project
```

---

## 📚 Documentation

| Document | Purpose | Read When |
|----------|---------|-----------|
| **QUICK_REFERENCE.md** | One-page cheat sheet | Daily use (print it!) |
| **SETUP_GUIDE.md** | Complete setup instructions | First time setup |
| **VISUAL_SUMMARY.md** | Architecture & data flow | Understanding how it works |
| **TOOL_STATUS_AND_USAGE.md** | Tool details & structure | Learning about components |

---

## 🔍 Real-World Example

### Your DeFi Protocol

```
contracts/
├── Token.sol           ← Main contract to verify
├── Vault.sol
└── Treasury.sol

spec/
├── Token.spec          ← CVL rules
├── Vault.spec
└── Treasury.spec

config.conf            ← Certora configuration
```

### Run Prover

```bash
certoractl run config.conf -o ./results
```

### List Failed Rules

```bash
$ cex-list ./results

Rule Name                    Status          CEX Available  
---------------------------------------------------------------
transfer_succeeds            VIOLATED        ✓ Yes          
balanceNeverNegative         VIOLATED        ✓ Yes          
reentrancySafe               VERIFIED        ✗ No           
```

### Analyze & Fix

```bash
# Analyze transfer_succeeds
cex-analyze ./results transfer_succeeds \
  --spec ./spec/Token.spec \
  --output transfer_analysis.md

# Read the analysis
cat transfer_analysis.md

# Apply suggested fix to contracts/Token.sol
# Re-run prover
certoractl run config.conf -o ./results_v2

# Verify it's now fixed
cex-list ./results_v2
# transfer_succeeds should now show: VERIFIED ✅
```

---

## ⚙️ Advanced Options

### Save to Different Output

```bash
cex-analyze ./results rule_name --output ~/my_analyses/analysis.md
```

### Use Custom API Key

```bash
cex-analyze ./results rule_name --api-key sk-ant-YOUR-KEY
```

### Extract Only (No Analysis)

```bash
cex-extract ./results rule_name --output cex_only.txt
```

### Just Display (No File)

```bash
cex-analyze ./results rule_name --spec spec.spec
# Output goes to terminal, not file
```

---

## 🐛 Troubleshooting

| Problem | Solution |
|---------|----------|
| "ANTHROPIC_API_KEY not set" | Run: `source ~/.certora-config` then check with `echo $ANTHROPIC_API_KEY` |
| "Rule not found" | Run `cex-list` first to see exact rule names |
| "No CEX found" | Only VIOLATED rules have CEX. Check the status in `cex-list` output |
| "Python not found" | Run: `source ~/certora-tools/venv/bin/activate` |
| "Permission denied" | Run: `chmod +x setup_certora_tools.sh` |
| "Module not found" | Reinstall: `pip install anthropic` |
| "Spec file not found" | Check path is correct relative to where you run the command |

---

## 💡 Pro Tips

### Tip 1: Git Track Your Analyses

```bash
git add analysis_*.md
git commit -m "CEX analyses for iteration 1"
```

Track progress over time and share with team.

### Tip 2: Compare Before/After Fixes

```bash
# Before fix
cex-analyze ./results_v1 rule_name --output before.md

# After fix
cex-analyze ./results_v2 rule_name --output after.md

# Compare
diff before.md after.md
```

### Tip 3: Generate HTML Report

```bash
# Convert markdown to HTML
pandoc analysis_*.md -o report.html

# Open in browser
open report.html
```

### Tip 4: Automate Everything

```bash
#!/bin/bash
# run_and_analyze.sh - Run prover and analyze all failures

certoractl run config.conf -o ./results

# Analyze all failures
cex-list ./results | grep VIOLATED | awk '{print $1}' | while read rule; do
    cex-analyze ./results "$rule" \
      --spec ./spec/*.spec \
      --output "analysis_${rule}.md"
done

echo "✅ Done! Review analysis_*.md"
```

---

## ✨ What Makes This Tool Different

| Aspect | Before (Manual) | After (This Tool) |
|--------|-----------------|-------------------|
| **Extract CEX** | Go to web UI, click around | 1 command: `cex-extract` |
| **Analyze CEX** | Copy, paste into Claude, wait | 1 command: `cex-analyze` |
| **Get fixes** | Manually interpret response | Formatted markdown report |
| **Multiple rules** | 1 at a time, ~5 min each | Batch all at once |
| **Total time for 5 rules** | ~30 minutes | ~5 minutes |
| **Reproducibility** | Lost when you close the browser | Saved to files, tracked in git |

---

## 📋 Before You Start

✅ **Checklist:**
- [ ] Python 3.9+ installed
- [ ] Anthropic API key obtained
- [ ] `setup_certora_tools.sh` downloaded
- [ ] Setup script executed
- [ ] Shell reloaded (`source ~/.bashrc`)
- [ ] `certora-verify` runs successfully

✅ **For Each Project:**
- [ ] `.certora-project` file created
- [ ] `config.conf` exists
- [ ] `./spec/` directory exists
- [ ] `./contracts/` directory exists

---

## 🔗 Resources

- **Get API Key:** https://console.anthropic.com/account/keys
- **Certora Docs:** https://docs.certora.com
- **Claude Docs:** https://docs.anthropic.com
- **GitHub Repo:** https://github.com/0xbrett8571/CertoraAIComposer

---

## 🚀 Next Steps

### First Time?
1. Run `./setup_certora_tools.sh`
2. Reload shell: `source ~/.bashrc`
3. Verify: `certora-verify`

### Have a Project?
1. Create `.certora-project` in project root
2. Run: `certoractl run config.conf -o ./results`
3. Analyze: `cex-analyze ./results rule_name --spec spec.spec`

### Need Help?
1. Check QUICK_REFERENCE.md for daily commands
2. See SETUP_GUIDE.md for detailed instructions
3. Review VISUAL_SUMMARY.md for architecture
4. Check troubleshooting section above

---

---

# Full Installation & AI Composer

For the original AI Composer (code generation from specs), see below:

## Installation

### Requirements

You will need at least Python 3.12, Docker (with compose), a Claude API key, and the ability to build the documentation (see [here](https://github.com/Certora/Documentation/?tab=readme-ov-file#building-locally-with-docker)).

### One-time DB setup

You will need to provision the various Postgres databases used by AI Composer. Do this as follows:

1. cd into `scripts/`
2. run `docker compose create && docker compose start`. This will initialize a local postgres database. NB: no attempt has been made to ensure this database is secure; caveat emptor

NB: You will need to restart this docker image each time your host computer restarts, unless you adjust the restart policy.

### One-time RAG Setup

You will need to build the local RAG database used for CVL manual searches by the LLM:

1. Run `./gen_docs.sh` to build the HTML documentation into `prover-docs/`
2. Run `./populate_rag.sh` to populate the standard `rag_db`

### One-time Extended RAG Setup (for Sanity Analyzer)

The sanity analyzer requires additional prover documentation beyond the CVL manual:

1. Run `./gen_docs.sh` (if not already done for the base RAG setup)
2. Run `./populate_extended_rag.sh` to populate the `extended_rag_db`

**Note:** The cex-analyzer and AI Composer use the standard `rag_db` (CVL-only), while sanity-analyzer defaults to `extended_rag_db` (CVL + prover docs). You can override this with the `--rag-db` flag.

### Updating the RAG

The RAG is read-only at runtime and is fully derived from the documentation HTML, so when the docs change you just rebuild it.
`./refresh_rag.sh` runs the full offline wipe+rebuild in one step (regenerate docs → wipe → repopulate), saving you from
chaining `gen_docs.sh`, `wipe_rag.py`, and the `populate_*.sh` scripts by hand:

```
./refresh_rag.sh                 # regenerate docs, then wipe + rebuild rag_db (CVL-only)
./refresh_rag.sh --all           # also wipe + rebuild extended_rag_db
./refresh_rag.sh --skip-gen-docs # rebuild from the HTML already in prover-docs/
```

**Run this offline** (when nothing is querying the RAG): it empties the target database and then re-embeds over a few
minutes, during which CVL manual search returns no results. See `./refresh_rag.sh --help` for all options.

### One-time prover setup

From the root of the Certora Prover repo, run `./gradlew copy-assets`. Ensure that your `CERTORA` environment
variable is configured to point to the output of this build (`CertoraProver/target`)

### AI Composer Requirements

Install the requirements for AI Composer via `uv sync --extra ml`. You may do this in
a virtual environment, and in such case you also need to install the dependencies for the `certora-cli`:
`uv pip install -r certora_cli_requirements.txt` from the `CertoraProver/scripts` folder, and optionally the Solidity compiler, if none is
available system-wide. Also be sure to activate this new virtual environment each time you want to run AI Composer.

### Solidity Compilers

AI Composer assumes that the solidity compiler is available on your `$PATH` and follows the naming convention `solcX.Y`, where `X` and `Y`
are taken from the Solidity version numbers: `0.X.Y`. For example, to make solc version 0.8.29 available to AI Composer, you must ensure
that an executable `solc8.29` is somewhere on your path. Currently the LLM is prompted to use solidity version 0.8.29 but you can adjust
the prompts as needed.

## Usage

AI Composer is primarily a command line tool, with some more graphical debugging utilities available for use.

### Basic Operation

Once you have completed the above setup, you can run AI Composer via:

```
python3 ./main.py cvl_input.spec interface_file.sol system_doc.txt
```

Where `cvl_input.spec` is the CVL specification the implementation must conform to, `interface_file.sol`
contains an `interface` definition which the generated contract must implement, and `system_doc.txt`
is a text file containing a description of the overall system (defining key concepts, etc.)

AI Composer will iterate some number of times while it attempts to generate code. This process is _semi_ automatic;
AI Composer may ask for help via the human in the loop tool, propose spec changes, or ask for requirement relaxation.
It is recommended that you "babysit" the process as it runs.

A basic trace of what the tool is doing is displayed to stdout. You can enable `--debug` to see _very_ verbose output, but
more friendly debugging options are described below.

Once generation is completed, the generated sources and the LLM commentary is dumped to stdout.

### Basic Options

A few options can help tweak your experience:

- `--prover-capture-output false` will have the prover runs invoked by the AI Composer print its output to stdout/stderr instead of being captured
- `--prover-keep-folders` will print the temporary directories used for the prover runs, and not clean them up
- `--debug-prompt-override PROMPT` will append whatever text you provide in `PROMPT` to the initial prompt. Useful for instructing the LLM to do different things
- `--tokens T` How many tokens to sample from the LLM. This needs to be _relatively_ high due to the amount of code that needs to be generated
- `--thinking-tokens T` how many tokens of the overall token budget should be used for thinking
- `--model` The name of the Anthropic model to use for the task. Defaults to sonnet
- `--thread-id` and `--checkpoint-id` are used for resuming workflows that crash or need tweaking (see below)
- `--summarization-threshold` enables the summarization of older messages after a certain threshold

### Resuming Workflows

The `--thread-id` and `--checkpoint-id` options allow you to resume AI Composer execution from a specific point in time. Together, these identifiers describe a checkpoint in the execution history.

**Thread ID**: Identifies a specific execution session of AI Composer. This is displayed early in the output when starting a workflow:

```
Selected thread id: crypto_session_6511ace2-cfbf-11f0-aeb6-e8cf83d12a2d
```

**Checkpoint ID**: Identifies a specific point within that session. This is displayed throughout execution as the workflow progresses:

```
current checkpoint: 1f0cfbf9-bbd9-6365-8001-90d0fca3dbdf
```

To resume from a specific checkpoint, provide both identifiers:

```
python3 ./main.py --thread-id crypto_session_6511ace2-cfbf-11f0-aeb6-e8cf83d12a2d --checkpoint-id 1f0cfbf9-bbd9-6365-8001-90d0fca3dbdf cvl_input.spec interface_file.sol system_doc.txt
```

This will restart execution from exactly that point in the workflow. NB the checkpoint ID does _not_ need to be the most recent; you can "time travel" if you decide
you dislike a decision you made previously.

### Debugging Options

#### Debug Console

During execution, you can pause the current workflow by sending SIGINT (usually by hitting Ctrl+C). Once the workflow reaches a
point of quiescence, you will be dropped into the "Debug Console". This console allows you to explore the current state of the implementation,
and review the entire message history. You can also use this console to provide explicit guidance; this guidance is echoed to the LLM verbatim.

The message history does NOT preserve messages across summarization boundaries.

#### Trace Visualizer

After completion of a session, if you wish to see a visualization of the entire process you can use the `traceDump.py` script.

The basic usage is:

```
python3 scripts/traceDump.py thread-id conn-string out-file
```

Where `thread-id` is the thread ID for the session you wish to visualize. `conn-string` is the PostgreSQL string for connecting to the audit database, this should be
`postgresql://audit_db_user:audit_db_password@localhost:5432/audit_db` unless you have changed where audit data is stored. `out-file` is the name of an HTML file into
which the visual will be dumped.

#### Exporting the Output

To get the final deliverable from AI Composer, use the VFS materializer like so:

```
python3 ./resume.py materialize thread-id path
```

where `thread-id` is the thread ID of the session whose output you wish to view, and `path` the path to a directory into which the resulting VFS is dumped.

### Meta-Iteration

Once AI Composer finishes generation, you can refine/adjust the specification and resume generation, seeding the process
with the output of a prior session. This is referred to as "meta-iteration".

Meta iteration can be done in one of two ways:

- use `materialize` command of `resume.py` (described above) to materialize the result of a prior run into a folder,
  arbitrarily changing the contents of that folder, and then using the `resume-dir` command of `resume.py`, OR
- using `resume-id` with the thread ID of a completed run and passing in an updated specification file

In the former case, the invocation looks like this:

```
python3 resume.py resume-dir thread-id path
```

Here `thread-id` is the thread ID of the workflow whose contents were materialized into `path`, the directory containing the changed
project files.

In the latter case, the invocation is:

```
python3 resume.py resume-id thread-id new-spec
```

where `thread-id` is the thread ID of the workflow on which you want to iterate, and `new-spec` is the path
to the updated/refined spec file to use for the next iteration.

---

# Disclaimer

AI Composer is a research prototype released by Certora Labs. The code generated by AI Composer should **not** be
placed into production without thorough vetting/testing/auditing.

The CEX Analysis Tool is a standalone utility that extracts and analyzes counterexamples from Certora Prover output
using Claude AI. While it provides actionable insights, suggested fixes should be reviewed and tested before deployment.
