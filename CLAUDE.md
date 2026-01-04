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

### Skills System

Skills are reusable multi-agent workflow templates stored as Neo4j nodes with:
- `triggers`: Keywords that auto-detect the skill
- `workflow_template`: JSON string defining delegation steps
- `parameters`: JSON string defining extractable parameters

Skill execution returns a delegation plan for main-agent to execute step-by-step.

## File Descriptions

- `main-agent` - Orchestrator prompt with routing logic, file-metadata detection, and response formatting rules
- `neo4j-management-agent` - Neo4j operations using parameterized Python scripts with argparse
- `cybersecurity-agent` - Kali MCP command execution with mandatory verification pattern
- `research-analysis-agent` - Web search agent (same content as cybersecurity-agent in current state)
- `files-retrieving-agent` - Qdrant vector search with similarity scoring and blueprint isolation
- `skill-manager-agent` - Skill lifecycle management (create, detect, execute, list, update, delete)

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
