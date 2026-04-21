# AI Agent for Gossip Validation Verification

## Overview
Build a multi-agent system to validate that consensus client implementations correctly implement all gossip validation rules specified in consensus-specs.

## Agent Architecture

### 1. Spec Parser Agent
**Input:** consensus-specs p2p-interface.md files
**Output:** Structured JSON of all validation rules

**Tasks:**
- Parse all p2p-interface.md files (phase0, altair, bellatrix, capella, deneb, electra, fulu)
- Extract all [REJECT] and [IGNORE] conditions for each gossip topic:
  - beacon_block
  - beacon_aggregate_and_proof
  - beacon_attestation_{subnet_id}
  - voluntary_exit
  - proposer_slashing
  - attester_slashing
  - (and fork-specific topics like sync_committee_contribution_and_proof)
- Structure each rule with:
  - Topic name
  - Condition type (REJECT/IGNORE)
  - Condition description
  - Code references (e.g., "data.slot <= current_slot")
  - Fork applicability

**Output Format:**
```json
{
  "topics": {
    "beacon_block": {
      "validations": [
        {
          "type": "IGNORE",
          "description": "The block is not from a future slot",
          "condition": "signed_beacon_block.message.slot <= current_slot",
          "tolerance": "MAXIMUM_GOSSIP_CLOCK_DISPARITY",
          "fork": "phase0+",
          "spec_reference": "specs/phase0/p2p-interface.md#beacon_block"
        },
        ...
      ]
    }
  }
}
```

### 2. Client Scanner Agent (Per Client)
**Input:** Client repository + validation rules JSON
**Output:** Client implementation mapping

**Tasks:**
- Identify the gossip validation handler code in each client
- For each gossip topic, find the validation function
- Extract the validation logic implementation
- Map client code to validation rule structure

**Client-specific entry points to find:**

| Client | Language | Likely Path Pattern |
|--------|----------|---------------------|
| Lighthouse | Rust | `beacon_node/network/src/sync/network_context/gossip_verify.rs` or similar |
| Prysm | Go | `beacon-chain/p2p/gossip_*.go` or `beacon-chain/sync/validate_*.go` |
| Teku | Java | `networking/eth2/src/main/java/tech/pegasys/teku/networking/eth2/gossip/` |
| Nimbus | Nim | `beacon_chain/networking/eth2_network.nim` or `gossip_validation.nim` |
| Lodestar | TypeScript | `packages/beacon-node/src/chain/validation/` |

**Output Format:**
```json
{
  "client": "lighthouse",
  "version": "v5.1.0",
  "language": "rust",
  "gossip_handlers": {
    "beacon_block": {
      "file": "beacon_node/network/src/sync/network_context.rs",
      "function": "verify_block_for_gossip",
      "line_start": 234,
      "implementation": "...[code snippet]..."
    }
  }
}
```

### 3. Matcher Agent (Per Client + Topic)
**Input:** Spec validation rules + Client implementation
**Output:** Verification results

**Tasks:**
- For each validation rule from spec:
  - Search for corresponding implementation in client code
  - Verify semantic equivalence (using AI reasoning)
  - Check if validation is missing
  - Check if validation order is correct
  - Flag any deviations or extensions

**Analysis criteria:**
- ✅ **Implemented correctly**: Code clearly implements the validation
- ⚠️ **Possibly incorrect**: Code exists but logic seems different
- ❌ **Missing**: No corresponding validation found
- ℹ️ **Extra validation**: Client implements additional checks not in spec
- 🔄 **Ordering issue**: Validations exist but in wrong order (IGNORE before REJECT matters!)

**Output Format:**
```json
{
  "client": "lighthouse",
  "topic": "beacon_block",
  "analysis": [
    {
      "rule_id": "beacon_block_reject_1",
      "spec_condition": "The proposer signature is valid",
      "status": "implemented_correctly",
      "client_location": "file.rs:123",
      "confidence": 0.95,
      "notes": "Uses verify_block_signature() which matches spec"
    },
    {
      "rule_id": "beacon_block_ignore_2",
      "spec_condition": "The block is not from a future slot",
      "status": "possibly_incorrect",
      "client_location": "file.rs:456",
      "confidence": 0.6,
      "notes": "Uses different clock disparity value (300ms vs spec 500ms)",
      "diff": "..."
    }
  ]
}
```

### 4. Report Generator Agent
**Input:** All matcher results
**Output:** Human-readable report + actionable issues

**Tasks:**
- Aggregate results across all clients
- Identify common issues
- Generate comparison matrix
- Create detailed reports per client
- Generate GitHub issues (optional)

**Output Format:**
- Markdown report
- CSV export
- GitHub issues (if requested)

## Implementation Steps

### Step 1: Build Spec Parser
```bash
# Create a script that uses Claude API to parse specs
python agent_scripts/parse_specs.py \
  --specs-dir ./consensus-specs/specs \
  --output validation_rules.json
```

### Step 2: Clone Client Repos
```bash
mkdir client_repos
cd client_repos
git clone https://github.com/sigp/lighthouse
git clone https://github.com/prysmaticlabs/prysm
git clone https://github.com/Consensys/teku
git clone https://github.com/status-im/nimbus-eth2
git clone https://github.com/ChainSafe/lodestar
```

### Step 3: Run Client Scanner (per client)
```bash
python agent_scripts/scan_client.py \
  --client lighthouse \
  --repo ./client_repos/lighthouse \
  --rules validation_rules.json \
  --output lighthouse_implementation.json
```

### Step 4: Run Matcher Agent
```bash
python agent_scripts/match_and_verify.py \
  --rules validation_rules.json \
  --client-impl lighthouse_implementation.json \
  --output lighthouse_analysis.json
```

### Step 5: Generate Report
```bash
python agent_scripts/generate_report.py \
  --analyses "*_analysis.json" \
  --output validation_report.md
```

## Technical Implementation Details

### Using Claude API (Recommended)

**Why not just Claude Code / "clawdbot"?**
- Claude Code is designed for interactive development tasks
- For automated validation pipeline, better to use Claude API directly
- Can batch process, schedule regular runs, integrate with CI

**Architecture:**
```python
from anthropic import Anthropic

class SpecParserAgent:
    def __init__(self):
        self.client = Anthropic()

    def parse_p2p_spec(self, spec_file_content):
        prompt = f"""
        Parse the following p2p-interface specification and extract all
        gossip validation rules. For each rule, identify:
        1. Topic name
        2. Validation type (REJECT/IGNORE)
        3. Condition description
        4. Code/logic referenced

        Spec content:
        {spec_file_content}

        Return structured JSON.
        """

        response = self.client.messages.create(
            model="claude-opus-4",
            max_tokens=16000,
            messages=[{"role": "user", "content": prompt}]
        )
        return response.content
```

### Using Claude Code Agent SDK (Alternative)

If you want to use Claude Code's agentic capabilities:

```python
# Use the extended context and tool use capabilities
from claude_code_sdk import Agent, CodebaseContext

agent = Agent(
    name="gossip-validator",
    model="claude-opus-4",
    tools=["read_file", "search_code", "grep"]
)

# Load consensus-specs context
specs_context = CodebaseContext("/path/to/consensus-specs")

# Load client context
client_context = CodebaseContext("/path/to/lighthouse")

result = agent.run(
    task="Verify all beacon_block gossip validations are implemented",
    contexts=[specs_context, client_context]
)
```

## Deployment Options

### Option A: Local Script
- Run Python scripts locally with Claude API
- Good for one-off analysis
- Cost: ~$5-20 per full analysis run

### Option B: GitHub Action / CI
- Schedule weekly/monthly runs
- Automatically detect changes
- Create GitHub issues for problems
- Integrate with client repos

### Option C: Web Dashboard
- Build a web app showing validation status
- Real-time updates
- Community visibility
- More complex to build

## Estimated Costs

Using Claude API (Opus 4):
- Spec parsing: ~100K tokens = $0.75
- Client scanning per client: ~500K tokens = $3.75
- Matching per client: ~300K tokens = $2.25
- Report generation: ~50K tokens = $0.40

**Total per run: ~$35 for all 5 major clients**

Could optimize by using:
- Sonnet 4 for simpler tasks ($0.30 vs $1.50 per million tokens)
- Caching for repeated spec content
- Incremental analysis (only changed files)

**Optimized cost: ~$10-15 per run**

## Next Steps

1. **Prototype the Spec Parser**
   - Start with phase0 p2p-interface.md
   - Test extraction accuracy
   - Refine prompt and output format

2. **Test on One Client (Lighthouse)**
   - Manually identify gossip validation code
   - Run scanner agent
   - Evaluate matcher accuracy

3. **Expand to All Clients**
   - Automate the pipeline
   - Handle different languages/patterns

4. **Productionize**
   - Add error handling
   - Create monitoring
   - Set up regular runs

## Alternative: Simpler Approach

If the full multi-agent system is too complex initially, you could:

1. **Manual codebase hints**: Provide agent with pre-identified validation handler locations
2. **Interactive mode**: Use Claude Code interactively to check each client
3. **Focused scope**: Start with just beacon_block validation, not all topics

Would you like me to help you:
1. Write the spec parser agent code?
2. Set up the infrastructure for this?
3. Start with a simpler prototype first?
