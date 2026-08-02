# Google Antigravity Agent Workspace Specification

This document provides the definitive file and directory blueprint for configuring agentic development parameters at the project workspace level within Google Antigravity.

---

## 📁 Complete Directory Tree

Place this entire structural layout at the absolute root of your project codebase:

```text
your-project-root/
└── .agents/                         # Root of workspace agent customizations
    ├── AGENT.md                     # Master identity, orchestration, & prompt routing
    ├── config.json                  # Native layout registration blueprint
    ├── mcp_config.json              # Workspace-scoped Model Context Protocol servers
    ├── hooks.json                   # Event interception mapping & guardrails
    │
    ├── rules/                       # Natural language persistent context injections
    │   ├── style-guide.md           # Markdown file containing formatting guidelines
    │   └── security-policies.md     # Markdown file containing runtime security boundaries
    │
    ├── skills/                      # Model-Based Skills (Natural language workflows)
    │   └── db-migration/
    │       └── SKILL.md             # Functional triggers & procedural steps
    │
    ├── tools/                       # Code-Based Skills (Executable runtime binaries)
    │   └── custom-linter-tool/
    │       ├── tool.json            # Machine-readable input parameters declaration
    │       └── check.py             # Active Python execution script called by agent
    │
    ├── hooks/                       # Event-driven programmatic hooks
    │   └── prevent-secrets.py       # Intercepts writes to check for exposed API tokens
    │
    └── knowledge/                   # RAG context injections (Static enterprise references)
        └── api-specs.json           # Raw target specifications data
```

---

## 🛠️ File Definitions, Details, and Examples

### 1. `.agents/config.json`
*   **Purpose:** The central manifest registration file. It explicitly directs the Antigravity parsing engine to the exact subdirectories containing your rules, skills, tools, and hooks.
*   **Implementation Example:**
```json
{
  "rules_path": "./rules",
  "skills_path": "./skills",
  "tools_path": "./tools",
  "knowledge_path": "./knowledge",
  "hooks_path": "./hooks"
}
```

### 2. `.agents/mcp_config.json`
*   **Purpose:** Declares local Model Context Protocol (MCP) servers. This connects external APIs, databases, or enterprise systems securely to the workspace context window.
*   **Implementation Example:**
```json
{
  "mcpServers": {
    "postgresql-inspector": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost:5432/dev_db"]
    }
  }
}
```

### 3. `.agents/hooks.json`
*   **Purpose:** Connects operational lifecycle events (like file writes or terminal updates) directly to your local validator script pathways.
*   **Implementation Example:**
```json
{
  "hooks": [
    {
      "event": "before_file_write",
      "action": "execute_script",
      "script_path": "./hooks/prevent-secrets.py"
    }
  ]
}
```

### 4. `.agents/AGENT.md`
*   **Purpose:** The baseline persona definition sheet. It tells the agent its technical level, background context, and maps overall file-routing architecture expectations.
*   **Implementation Example:**
```markdown
---
name: "Senior Core Engineer"
description: "Master runtime architect for this workspace repository"
---

# Workspace Identity
You are an expert full-stack developer optimizing a modern web application workspace. 

# Core Task Execution Parameters
- Always query the `./rules/` directory before proposing file changes.
- Prioritize test files alongside code generations.
```

### 5. `.agents/rules/style-guide.md`
*   **Purpose:** Houses layout restrictions or structural formatting requirements. Frontmatter controls when the rule applies to your files.
*   **Implementation Example:**
```markdown
---
description: "TypeScript and code formatting guidelines"
activation: glob
pattern: "src/**/*.ts"
---

# Styling Guardrails
- Use explicitly typed parameters; completely avoid `any`.
- Enforce camelCase formatting conventions for all local variable declarations.
```

### 6. `.agents/skills/db-migration/SKILL.md`
*   **Purpose:** Provides structural, natural-language operational recipes for complex sequence operations using the agent's built-in platform skills.
*   **Implementation Example:**
```markdown
---
name: "apply-database-migrations"
description: "Executes structural database version updates safely"
triggers:
  - "when schema modifications occur"
---

# Operational Instructions
1. Run `npm run db:status` to verify pending states.
2. If schema updates exist, execute `npm run db:migrate`.
```

### 7. `.agents/tools/custom-linter-tool/tool.json`
*   **Purpose:** A structural schema definition file that registers an underlying script executable to the engine as an automated tool.
*   **Implementation Example:**
```json
{
  "name": "custom_file_linter",
  "description": "Executes a Python linter script across a target file",
  "input_schema": {
    "type": "object",
    "properties": {
      "file_path": { "type": "string", "description": "Path to file being processed" }
    },
    "required": ["file_path"]
  }
}
```

### 8. `.agents/tools/custom-linter-tool/check.py`
*   **Purpose:** The physical programmatic execution file assigned to the tool specification blueprint above.
*   **Implementation Example:**
```python
import sys
import json

def process_file():
    args = json.load(sys.stdin)
    path = args.get("file_path")
    # File evaluation computations go here
    print(json.dumps({"status": "success", "message": f"Verified {path}"}))

if __name__ == "__main__":
    process_file()
```

### 9. `.agents/hooks/prevent-secrets.py`
*   **Purpose:** A low-level security interception hook script that parses proposed context updates over standard input loops before modifications are applied to your active project files.
*   **Implementation Example:**
```python
import sys
import json

def main():
    payload = json.load(sys.stdin)
    proposed_content = payload.get("content", "")
    
    if "sk_live_" in proposed_content:
        # Blocks execution if security parameter is broken
        print(json.dumps({"action": "deny", "reason": "Secret API token pattern matched."}))
    else:
        print(json.dumps({"action": "allow"}))

if __name__ == "__main__":
    main()
```

### 10. `.agents/knowledge/api-specs.json`
*   **Purpose:** Static data payloads stored in the repository to provide clean retrieval documentation without inflating prompt tokens.
*   **Implementation Example:**
```json
{
  "endpoint": "/api/v2/users",
  "allowed_methods": ["GET", "POST"],
  "response_format": "application/json"
}
```
