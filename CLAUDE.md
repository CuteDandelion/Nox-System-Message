# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This repository maintains the system messages (prompts) for NOX, a multi-agent AI orchestration system. NOX consists of a main orchestrator agent that delegates tasks to specialized sub-agents.

## Architecture Overview

### Agent Hierarchy

```
main-agent (Pure Orchestrator)
├── neo4j-graph-management-agent   (Graph DB operations via Python + Neo4j driver)
├── research-analysis-agent        (Web search, CVE research)
├── cybersecurity-agent            (Command execution via Kali MCP)
├── files-retrieving-agent         (Document retrieval from Qdrant vector DB)
└── skill-manager-agent            (Reusable workflow templates stored in Neo4j)
```

### Key Design Principles

1. **Zero-Knowledge Orchestration**: The main-agent has no domain knowledge - it only routes tasks to sub-agents and presents results
2. **Blueprint Isolation**: Data is isolated by `blueprintId` (default: `Blueprint#1`) - queries must always filter by this
3. **Mandatory Delegation**: Main-agent must delegate all technical work; generating responses from memory is considered "hallucination"
4. **Transparency**: Neo4j and skill operations must show full Python scripts, commands executed, and outputs

### Communication Protocol

Inter-agent messages use this format:
```
[TASK-ID: task-001]
[BLUEPRINT: Blueprint#1]
[FROM: agent-name]
[TO: agent-name]

[Task content]

Context: [Why]
Expected output: [Format]
```

### File Type Routing (main-agent)

When `file-metadata_*` is detected in user message:
- `type = image/*` → main-agent handles directly (images not in Qdrant)
- All other types → delegate to files-retrieving-agent (even if content visible in context)

### Sub-Agent Tool Mappings

| Agent | MCP Tool | Purpose |
|-------|----------|---------|
| neo4j-graph-management-agent | Neo4j-ExecutePythonQuery | Graph operations via Python + Neo4j driver |
| cybersecurity-agent | kali_mcp:execute_command | Bash/script execution on Kali Linux |
| research-analysis-agent | web-search MCP | Web research, CVE lookup |
| files-retrieving-agent | qdrant-retrieve | Vector DB document retrieval |
| skill-manager-agent | Neo4j-ExecutePythonQuery | Skill CRUD in Neo4j |

### Skills System (V5 - Disk-Based Metadata JSON)

**Key change in V5:** Main-agent writes ALL files to disk including a `metadata.json` file. Skill-manager-agent reads from the JSON file (no embedded JSON in scripts - this fixes escaping bugs).

**The V4 "dictionary update sequence" error was caused by:** Embedding JSON in Python scripts led to escaping corruption. V5 solves this by reading JSON from disk with `json.load()`.

Workflow:
1. User requests skill creation
2. **Main-agent** plans the workflow, shows user for approval
3. User approves (says "yes", "create", "do it", etc.)
4. **PHASE 1 - Write Files** (via cybersecurity-agent):
   - Step scripts: `/tmp/skill_<name>_step_N.py` or `.sh`
   - Metadata JSON: `/tmp/skill_<name>_metadata.json`
   - Report: "✓ Files created successfully"
5. **PHASE 2 - Store Skill** (via skill-manager-agent):
   - Send metadata file path
   - skill-agent reads JSON, stores in Neo4j
   - Report: "✓ Skill stored with ID: skill-xxx"
6. Final confirmation: "✅ Skill 'Name' created successfully!"

**CRITICAL:** Both phases happen automatically after approval - don't stop after Phase 1!

**Metadata JSON file structure:**
```json
{
  "name": "Network Scanner",
  "description": "Scans network and stores hosts",
  "category": "Security",
  "triggers": ["scan network", "network discovery"],
  "parameters": {
    "target_network": {"type": "string", "required": true}
  },
  "workflow_template": {
    "steps": [
      {
        "step": 1,
        "agent": "cybersecurity-agent",
        "filename": "/tmp/skill_network_scanner_step_1.sh",
        "command": "bash /tmp/skill_network_scanner_step_1.sh {{target_network}}",
        "description": "Scan network for live hosts",
        "params": ["target_network"],
        "timeout": 120
      },
      {
        "step": 2,
        "agent": "neo4j-graph-management-agent",
        "filename": "/tmp/skill_network_scanner_step_2.py",
        "command": "python3 /tmp/skill_network_scanner_step_2.py --hosts '{{step_1_output}}' --blueprint '{{blueprint_id}}' --timeout 60 --task-id '{{task_id}}'",
        "description": "Store hosts in Neo4j",
        "params": ["hosts", "blueprint_id"],
        "timeout": 60
      }
    ]
  }
}
```

**Key fields in each step:**
- `filename`: Path to script file on disk
- `command`: **EXACT command to run** - sub-agents copy this, substitute `{{placeholders}}`, execute
- `params`: Parameters referenced in command
- `output_var`: (optional) Store output for use in later steps

**Available placeholders:**
- `{{param_name}}` - User-provided parameter
- `{{step_N_output}}` - Output from step N
- `{{blueprint_id}}` - Current blueprint (default: Blueprint#1)
- `{{task_id}}` - Current task ID

**Execution:** Main-agent reads `command` field, substitutes placeholders, executes:
```bash
# Original command in workflow_template:
# "command": "bash /tmp/skill_network_scanner_step_1.sh {{target_network}}"

# After substitution:
bash /tmp/skill_network_scanner_step_1.sh 192.168.1.0/24
```

**Why disk-based metadata + command field works:**
- `json.load(file)` NEVER has escaping issues
- All skill data in one place (metadata.json)
- Sub-agents don't need to figure out how to run scripts
- Updates just overwrite files - same flow as create

### Delete Operations Routing

- **Graph data** (nodes, relationships) → neo4j-graph-management-agent (Skills auto-protected)
- **Skills** → skill-manager-agent (only agent that can delete Skills)

## File Descriptions

- `main-agent` - Orchestrator prompt with routing logic, file-metadata detection, response formatting, and **skill planning** (V4)
- `neo4j-management-agent` - Neo4j operations using parameterized Python scripts, **delete operations with Skills protection** (V4)
- `cybersecurity-agent` - Kali MCP command execution with mandatory verification pattern
- `research-analysis-agent` - Web search agent (same content as cybersecurity-agent in current state)
- `files-retrieving-agent` - Qdrant vector search with similarity scoring and blueprint isolation
- `skill-manager-agent` - **Storage-only** skill agent (V4): stores pre-planned workflows, returns them for execution

## Common Patterns

### Neo4j Script Execution
```bash
cat > /tmp/neo4j_<timestamp>.py << 'EOF'
[PYTHON SCRIPT]
EOF
python3 /tmp/neo4j_<timestamp>.py --blueprint "Blueprint#1" --timeout 60 --task-id "task-123"
```

### Kali Command Verification (Bundled)
```bash
# Bundled create+verify in ONE command (prevents race conditions)
kali_mcp:execute_command("cat > /tmp/script.sh << 'EOF'
#!/bin/bash
# script content
EOF
chmod +x /tmp/script.sh && test -f /tmp/script.sh && test -x /tmp/script.sh && echo 'VERIFIED' || echo 'FAILED'")

# For Python files (no chmod +x needed):
kali_mcp:execute_command("cat > /tmp/script.py << 'EOF'
#!/usr/bin/env python3
# script content
EOF
test -f /tmp/script.py && echo 'VERIFIED' || echo 'FAILED'")

# For JSON files:
kali_mcp:execute_command("cat > /tmp/metadata.json << 'EOF'
{"name": "example", "value": 123}
EOF
test -f /tmp/metadata.json && echo 'VERIFIED' || echo 'FAILED'")
```

**CRITICAL - `cat >` vs `cat`:**
- `cat > /tmp/file << 'EOF'` = **WRITE** (creates file with heredoc content)
- `cat /tmp/file` = **READ** (prints file content, FAILS if file doesn't exist)
- ALWAYS use `cat >` with `>` redirect to CREATE files!

## When Editing These Prompts

- Preserve the `[TASK-ID]`, `[STATUS]`, `[FROM]`, `[TO]`, `[BLUEPRINT]` tag structure
- Maintain blueprint isolation - all queries must filter by `blueprintId`
- Keep transparency requirements for neo4j-agent and skill-manager-agent (show full scripts)
- Respect the single-execution rule for cybersecurity-agent (max 2 tool calls: command + verify)
- Test routing logic changes against the file-metadata detection flow in main-agent
- **V5 skill workflow**: main-agent plans skills, writes ALL files (steps + metadata.json), skill-agent reads from disk
- **V5 delete routing**: neo4j-agent handles deletes BUT must protect Skills; skill-agent handles skill deletion
- **V5 metadata pattern**: skill-agent receives file path, reads with `json.load()` - no embedded JSON in scripts

## V5 Changes Summary

Key changes in version 5:

1. **Disk-based metadata JSON**: Main-agent writes a `metadata.json` file containing ALL skill info (name, triggers, parameters, workflow_template). Skill-manager-agent reads from this file.

2. **Fixes "dictionary update sequence" error**: The V4 approach of embedding JSON in Python scripts caused escaping corruption. V5 uses `json.load(file)` which never has escaping issues.

3. **Full skill creation after approval**: When user approves (says "yes", "create", etc.), main-agent automatically: (1) writes ALL files, (2) calls skill-manager-agent to store in Neo4j, (3) reports completion. Both phases happen automatically.

4. **Simpler skill-agent protocol**: Main-agent sends just the metadata file path. Skill-agent reads JSON from disk, stores in Neo4j. No parsing embedded strings.

5. **Consistent update flow**: Updates just overwrite the files and call skill-agent with the same metadata file path. Same flow as create.

6. **Skills protection in neo4j-agent**: All delete operations automatically exclude Skills with `WHERE NOT n:Skill`. Only skill-manager-agent can delete Skills.

7. **Command field in workflow steps**: Each step now includes a `command` field with the EXACT command to run (e.g., `python3 /tmp/script.py --param '{{value}}' --timeout 60`). Sub-agents just substitute placeholders and execute - no figuring out how to run scripts.

8. **Placeholder substitution**: Commands use `{{placeholder}}` syntax for parameters (`{{target_network}}`), previous step output (`{{step_1_output}}`), and system values (`{{blueprint_id}}`, `{{task_id}}`).
