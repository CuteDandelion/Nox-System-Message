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

### Skills System (V4 - Main-Agent Plans, Skill-Agent Stores)

**Key change in V4:** Main-agent now plans and writes skill workflows (including scripts). Skill-manager-agent only stores and retrieves them.

Workflow:
1. User requests skill creation
2. **Main-agent** plans the workflow, writes Python scripts/bash commands, includes filenames
3. User approves the plan
4. Main-agent sends COMPLETE package to skill-manager-agent
5. **Skill-manager-agent** stores it exactly as received (no modification)

Skill nodes contain:
- `triggers`: Keywords that auto-detect the skill
- `workflow_template`: JSON string with steps, each containing:
  - `agent`: Which sub-agent executes this step
  - `type`: "script" or "command"
  - `filename`: **REQUIRED** - path where script will be written (e.g., `/tmp/skill_name_step_1.py`)
  - `content`/`command`: The actual code/command
  - `timeout`: Execution timeout
- `parameters`: JSON string defining extractable parameters

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

### Kali Command Verification
```bash
# Execute
kali_mcp:execute_command("cat > /tmp/script.sh << 'EOF'...")
# Verify
kali_mcp:execute_command("test -f /tmp/script.sh && echo 'VERIFIED' || echo 'FAILED'")
```

## When Editing These Prompts

- Preserve the `[TASK-ID]`, `[STATUS]`, `[FROM]`, `[TO]`, `[BLUEPRINT]` tag structure
- Maintain blueprint isolation - all queries must filter by `blueprintId`
- Keep transparency requirements for neo4j-agent and skill-manager-agent (show full scripts)
- Respect the single-execution rule for cybersecurity-agent (max 2 tool calls: command + verify)
- Test routing logic changes against the file-metadata detection flow in main-agent
- **V4 skill workflow**: main-agent plans skills (writes code), skill-agent only stores
- **V4 delete routing**: neo4j-agent handles deletes BUT must protect Skills; skill-agent handles skill deletion

## V4 Changes Summary

Key changes in version 4:

1. **Skill planning moved to main-agent**: Main-agent now plans entire workflows including Python scripts and bash commands. Skill-manager-agent is purely a storage agent.

2. **Filename field required in skills**: Every step in `workflow_template` must include a `filename` field so main-agent knows where to write and execute scripts.

3. **Skills protection in neo4j-agent**: All delete operations in neo4j-management-agent automatically exclude Skills with `WHERE NOT n:Skill`. Only skill-manager-agent can delete Skills.

4. **Stronger anti-hallucination in skill-agent**: Skill-manager-agent must call Neo4j-ExecutePythonQuery IMMEDIATELY - no narration, no verification loops, ONE tool call per operation.
