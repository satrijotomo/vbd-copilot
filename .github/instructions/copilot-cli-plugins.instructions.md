---
applyTo: "**/plugin.json,**/plugins/**,**/.github/plugin/**"
---

# GitHub Copilot CLI Plugin Development — Complete Reference

Use this instruction set whenever you are asked to create, scaffold, or modify a Copilot CLI plugin. A plugin is an installable package that bundles agents, skills, hooks, and MCP server configurations into a single distributable unit.

## Plugin Directory Structure

Every plugin is a directory with a `plugin.json` manifest at the root. All other components are optional.

```text
my-plugin/
├── plugin.json              # REQUIRED — manifest (name, description, component paths)
├── agents/                  # Custom agents (*.agent.md files)
│   ├── reviewer.agent.md
│   └── planner.agent.md
├── skills/                  # Skills (each in its own subdirectory with SKILL.md)
│   ├── deploy/
│   │   ├── SKILL.md
│   │   └── deploy.sh       # Optional companion scripts
│   └── test-runner/
│       └── SKILL.md
├── hooks.json               # Hook configuration (lifecycle event handlers)
├── hooks/                   # Alternative: hooks directory with scripts
│   ├── hooks.json
│   ├── pre-tool-check.sh
│   └── post-tool-log.sh
├── .mcp.json                # MCP server configurations
└── lsp.json                 # LSP server configurations (rare)
```

## plugin.json — The Manifest

### Required Field

| Field  | Type   | Rules |
|--------|--------|-------|
| `name` | string | Kebab-case only (letters, numbers, hyphens). Max 64 chars. |

### Optional Metadata Fields

| Field         | Type     | Description |
|---------------|----------|-------------|
| `description` | string   | Brief description. Max 1024 chars. |
| `version`     | string   | Semver (e.g., `1.0.0`). |
| `author`      | object   | `{ "name": "...", "email": "...", "url": "..." }` (name required). |
| `homepage`    | string   | Plugin homepage URL. |
| `repository`  | string   | Source repository URL. |
| `license`     | string   | License identifier (e.g., `MIT`). |
| `keywords`    | string[] | Search keywords. |
| `category`    | string   | Plugin category. |
| `tags`        | string[] | Additional tags. |

### Component Path Fields

These tell the CLI where to find plugin components. All are optional — the CLI uses default conventions if omitted.

| Field        | Type              | Default     | Description |
|--------------|-------------------|-------------|-------------|
| `agents`     | string \| string[] | `agents/`   | Path(s) to directories containing `.agent.md` files. |
| `skills`     | string \| string[] | `skills/`   | Path(s) to directories containing skill subdirectories (each with `SKILL.md`). |
| `commands`   | string \| string[] | —           | Path(s) to command directories. |
| `hooks`      | string \| object   | —           | Path to a hooks config file, or inline hooks object. |
| `mcpServers` | string \| object   | —           | Path to an MCP config file (e.g., `.mcp.json`), or inline server definitions. |
| `lspServers` | string \| object   | —           | Path to an LSP config file, or inline server definitions. |

### Example plugin.json

```json
{
  "name": "my-dev-tools",
  "description": "Development utilities for React projects",
  "version": "1.0.0",
  "author": {
    "name": "Your Name",
    "email": "you@example.com"
  },
  "license": "MIT",
  "keywords": ["react", "frontend", "testing"],
  "agents": "agents/",
  "skills": ["skills/", "extra-skills/"],
  "hooks": "hooks.json",
  "mcpServers": ".mcp.json"
}
```

## Custom Agents (*.agent.md)

Agent files use YAML frontmatter + Markdown body. Place them in the `agents/` directory (or wherever `plugin.json` points).

### YAML Frontmatter Properties

| Property      | Type     | Required | Description |
|---------------|----------|----------|-------------|
| `name`        | string   | Yes      | Agent identifier (kebab-case recommended). |
| `description` | string   | Yes      | When to use this agent. Copilot uses this for inference routing. |
| `tools`       | string[] | No       | Tool allowlist. Omit or use `["*"]` for all tools. Use `[]` to disable all. |
| `infer`       | boolean  | No       | When `true`, Copilot auto-delegates matching tasks to this agent. Default: `true` for CLI. |
| `mcp-servers` | object   | No       | Inline MCP server definitions (YAML format). |

### Tool Aliases

Use these portable aliases in the `tools` list:

| Alias     | Maps to                            | Purpose |
|-----------|------------------------------------|---------|
| `execute` | `bash`, `shell`, `powershell`      | Run shell commands |
| `read`    | `view`, `Read`, `NotebookRead`     | Read file contents |
| `edit`    | `Edit`, `MultiEdit`, `Write`       | Edit files |
| `search`  | `Grep`, `Glob`                     | Search files and text |
| `agent`   | `custom-agent`, `Task`             | Invoke other agents |
| `web`     | `WebSearch`, `WebFetch`            | Web fetching |

### Example Agent File

```markdown
---
name: security-auditor
description: Performs security audits on code. Use when asked to check for vulnerabilities, review security, or audit code.
tools: ["read", "search", "execute"]
---

You are a security specialist. You analyze code for vulnerabilities including:

- SQL injection, XSS, CSRF
- Secrets or credentials in source code
- Insecure dependencies
- Authentication/authorization bypass
- Path traversal and input validation issues

## Process

1. Scan the target files or directories
2. Identify potential vulnerabilities with severity ratings
3. Provide specific remediation guidance for each finding
4. Create a summary report as a markdown file

## Output Format

For each finding:
- **Severity**: Critical / High / Medium / Low
- **File**: path and line number
- **Issue**: description of the vulnerability
- **Fix**: specific code change or mitigation
```

### Agent with MCP Server

```markdown
---
name: database-helper
description: Helps with database queries and schema management. Use when asked about databases, SQL, or data models.
tools: ["read", "edit", "search", "execute", "database-mcp/*"]
mcp-servers:
  database-mcp:
    type: local
    command: npx
    args: ["@modelcontextprotocol/server-sqlite", "--db", "./data/app.db"]
    tools: ["*"]
---

You are a database specialist that helps with SQL queries, schema design, and data management.
```

## Skills (SKILL.md)

Each skill lives in its own subdirectory under the skills directory. The directory must contain a `SKILL.md` file. Optional companion scripts and resources can be placed alongside it.

### SKILL.md Frontmatter

| Property      | Type   | Required | Description |
|---------------|--------|----------|-------------|
| `name`        | string | Yes      | Unique identifier (kebab-case, typically matches directory name). |
| `description` | string | Yes      | What the skill does and when to use it. Copilot uses this for auto-selection. |
| `license`     | string | No       | License identifier. |

### Example Skill

Directory: `skills/code-review/`

```markdown
---
name: code-review
description: Performs thorough code reviews with focus on correctness, performance, and maintainability. Use when asked to review code, check a PR, or audit changes.
---

# Code Review

You perform detailed code reviews focusing on:

## Review Checklist

1. **Correctness** - Does the code do what it claims?
2. **Edge cases** - Are boundary conditions handled?
3. **Performance** - Any N+1 queries, unnecessary allocations, or blocking calls?
4. **Security** - Input validation, auth checks, data sanitization?
5. **Maintainability** - Clear naming, reasonable complexity, adequate tests?

## Output Format

Use this structure for each finding:

- 🔴 **Bug**: Must fix before merge
- 🟡 **Concern**: Should address, not blocking
- 🟢 **Suggestion**: Nice to have improvement
- 💡 **Note**: FYI, no action needed

## Process

1. Read the changed files
2. Understand the intent from commit messages or PR description
3. Review each file systematically against the checklist
4. Write findings grouped by file, ordered by severity
```

### Skill with Companion Scripts

Directory: `skills/deploy/`

```text
skills/deploy/
├── SKILL.md
├── deploy.sh
├── validate-env.sh
└── rollback.sh
```

```markdown
---
name: deploy
description: Deploy the current project to staging or production. Use when asked to deploy, release, or push to an environment.
---

# Deploy Skill

## Available Scripts

- `deploy.sh` - Main deployment script. Run with: `bash skills/deploy/deploy.sh <environment>`
- `validate-env.sh` - Validates environment configuration before deploying
- `rollback.sh` - Rolls back to the previous deployment

## Process

1. Run `validate-env.sh` to check prerequisites
2. Confirm the target environment with the user
3. Run `deploy.sh <environment>` to deploy
4. Verify deployment succeeded
5. If deployment fails, run `rollback.sh` and report the error
```

## Hooks (hooks.json)

Hooks execute shell commands at specific lifecycle moments. They can be defined as a separate file or inline in `plugin.json`.

### Hook Types

| Hook                   | When it fires                      | Can block? |
|------------------------|------------------------------------|------------|
| `sessionStart`         | New or resumed session begins      | No         |
| `sessionEnd`           | Session completes or is terminated | No         |
| `userPromptSubmitted`  | User submits a prompt              | No         |
| `preToolUse`           | Before any tool executes           | Yes — can deny |
| `postToolUse`          | After a tool completes             | No         |
| `errorOccurred`        | When an error occurs               | No         |

### Hook Entry Properties

| Property      | Type   | Required | Description |
|---------------|--------|----------|-------------|
| `type`        | string | Yes      | Always `"command"`. |
| `bash`        | string | Yes*     | Shell command or script path (Unix). |
| `powershell`  | string | No       | PowerShell equivalent (Windows). |
| `cwd`         | string | No       | Working directory for the command. |
| `timeoutSec`  | number | No       | Timeout in seconds (default: 30). |
| `env`         | object | No       | Environment variables as key-value pairs. |
| `comment`     | string | No       | Human-readable description of the hook. |

### preToolUse Output (the only hook that can block)

The `preToolUse` hook can output JSON to allow or deny tool execution:

```json
{
  "permissionDecision": "deny",
  "permissionDecisionReason": "Dangerous command detected"
}
```

Valid `permissionDecision` values: `"allow"`, `"deny"`, `"ask"` (only `"deny"` is currently processed).

### Example hooks.json

```json
{
  "version": 1,
  "hooks": {
    "sessionStart": [
      {
        "type": "command",
        "bash": "echo \"Session started: $(date)\" >> logs/session.log",
        "timeoutSec": 10,
        "comment": "Log session start"
      }
    ],
    "preToolUse": [
      {
        "type": "command",
        "bash": "./hooks/security-check.sh",
        "comment": "Block dangerous commands"
      }
    ],
    "postToolUse": [
      {
        "type": "command",
        "bash": "./hooks/audit-log.sh",
        "comment": "Log all tool usage"
      }
    ],
    "sessionEnd": [
      {
        "type": "command",
        "bash": "echo \"Session ended: $(date)\" >> logs/session.log",
        "timeoutSec": 10,
        "comment": "Log session end"
      }
    ]
  }
}
```

### Example preToolUse Script

```bash
#!/bin/bash
INPUT=$(cat)
TOOL_NAME=$(echo "$INPUT" | jq -r '.toolName')
TOOL_ARGS=$(echo "$INPUT" | jq -r '.toolArgs')

# Block destructive commands
if echo "$TOOL_ARGS" | grep -qE "rm -rf /|DROP TABLE|format "; then
  echo '{"permissionDecision":"deny","permissionDecisionReason":"Destructive command blocked by plugin policy"}'
  exit 0
fi

# Allow everything else
echo '{"permissionDecision":"allow"}'
```

### Hook Input JSON Reference

**preToolUse input:**
```json
{
  "timestamp": 1704614600000,
  "cwd": "/path/to/project",
  "toolName": "bash",
  "toolArgs": "{\"command\":\"npm test\",\"description\":\"Run tests\"}"
}
```

**postToolUse input:**
```json
{
  "timestamp": 1704614700000,
  "cwd": "/path/to/project",
  "toolName": "bash",
  "toolArgs": "{\"command\":\"npm test\"}",
  "toolResult": {
    "resultType": "success",
    "textResultForLlm": "All tests passed (15/15)"
  }
}
```

**sessionStart input:**
```json
{
  "timestamp": 1704614400000,
  "cwd": "/path/to/project",
  "source": "new",
  "initialPrompt": "Fix the bug"
}
```

## MCP Server Configuration (.mcp.json)

Define MCP servers to give your plugin access to external tools and data sources.

### Server Types

| Type    | Transport   | Description |
|---------|-------------|-------------|
| `local` / `stdio` | stdin/stdout | Spawns a local process |
| `http`  | Streamable HTTP | Remote HTTP server |
| `sse`   | Server-Sent Events | Legacy remote server (deprecated in MCP spec) |

### Example .mcp.json

```json
{
  "mcpServers": {
    "filesystem": {
      "type": "local",
      "command": "npx",
      "args": ["@modelcontextprotocol/server-filesystem", "/path/to/allowed/dir"],
      "env": {},
      "tools": ["*"]
    },
    "github": {
      "type": "local",
      "command": "npx",
      "args": ["@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      },
      "tools": ["*"]
    },
    "remote-api": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${API_KEY}"
      },
      "tools": ["*"]
    }
  }
}
```

## Precedence Rules

When multiple plugins define components with the same name:

| Component         | Rule              | Explanation |
|-------------------|-------------------|-------------|
| Agents & Skills   | First-found wins  | Project-level > user-level > plugin (by install order). Plugins cannot override project or personal config. |
| MCP Servers       | Last-wins          | Later-loaded configs override earlier ones. `--additional-mcp-config` has highest priority. |
| Built-in tools    | Cannot be overridden | `bash`, `view`, `edit`, `search`, `task`, etc. are always present. |

Agent dedup uses the file name (e.g., `reviewer.agent.md` -> ID `reviewer`). Skill dedup uses the `name` field in `SKILL.md`.

## Loading Order (high to low priority for agents/skills)

1. `~/.copilot/agents/` and `~/.copilot/skills/` (user-level)
2. `<project>/.github/agents/` and `<project>/.github/skills/` (project-level)
3. Parent directories' `.github/agents/` and `.github/skills/` (monorepo inheritance)
4. Plugin components (by install order)
5. Remote org/enterprise agents

## Development Workflow

### 1. Scaffold the plugin

Create the directory structure and `plugin.json` manifest. Start with the minimum required field (`name`) and add components incrementally.

### 2. Install locally for testing

```bash
copilot plugin install ./my-plugin
```

### 3. Verify installation

```bash
copilot plugin list
```

In an interactive session:
```
/plugin list
/agent           # Check custom agents loaded
/skills list     # Check skills loaded
/mcp show        # Check MCP servers loaded
```

### 4. Iterate

After making changes to a local plugin, reinstall to pick up changes (components are cached):

```bash
copilot plugin install ./my-plugin
```

### 5. Uninstall

```bash
copilot plugin uninstall my-plugin
```

Use the plugin `name` from `plugin.json`, not the directory path.

## Distribution

### Direct install from GitHub

Users install directly from a repository:

```bash
copilot plugin install owner/repo
```

The repo must contain `plugin.json` at the root, in `.github/plugin/`, or in `.claude-plugin/`.

For plugins in a subdirectory:

```bash
copilot plugin install owner/repo:path/to/plugin
```

### Marketplace distribution

Create a `marketplace.json` in `.github/plugin/` of a repository:

```json
{
  "name": "my-marketplace",
  "owner": {
    "name": "Your Organization",
    "email": "plugins@example.com"
  },
  "metadata": {
    "description": "Curated plugins for our team",
    "version": "1.0.0"
  },
  "plugins": [
    {
      "name": "security-tools",
      "description": "Security auditing and enforcement",
      "version": "1.0.0",
      "source": "./plugins/security-tools"
    },
    {
      "name": "deploy-helpers",
      "description": "Deployment automation utilities",
      "version": "2.0.0",
      "source": "./plugins/deploy-helpers"
    }
  ]
}
```

Users register and install:

```bash
copilot plugin marketplace add owner/repo
copilot plugin install security-tools@my-marketplace
```

## Plugin File Locations

| Item                | Path |
|---------------------|------|
| Installed plugins   | `~/.copilot/state/installed-plugins/MARKETPLACE/PLUGIN-NAME` (marketplace) or `~/.copilot/state/installed-plugins/PLUGIN-NAME` (direct) |
| Marketplace cache   | `~/.copilot/state/marketplace-cache/` |
| Plugin manifest     | `plugin.json`, `.github/plugin/plugin.json`, or `.claude-plugin/plugin.json` |
| Marketplace manifest| `.github/plugin/marketplace.json` or `.claude-plugin/marketplace.json` |

## Complete Plugin Examples

### Example: Full-Stack Review Plugin

```text
fullstack-review/
├── plugin.json
├── agents/
│   ├── frontend-reviewer.agent.md
│   ├── backend-reviewer.agent.md
│   └── api-reviewer.agent.md
├── skills/
│   └── review-checklist/
│       └── SKILL.md
└── hooks.json
```

**plugin.json:**
```json
{
  "name": "fullstack-review",
  "description": "Specialized code review agents for frontend, backend, and API layers",
  "version": "1.0.0",
  "author": { "name": "Your Team" },
  "license": "MIT",
  "keywords": ["review", "frontend", "backend", "api"]
}
```

**agents/frontend-reviewer.agent.md:**
```markdown
---
name: frontend-reviewer
description: Reviews frontend code for React best practices, accessibility, and performance. Use when reviewing UI components, pages, or frontend utilities.
tools: ["read", "search"]
---

You are a frontend code review specialist. Focus on:

- React component patterns (hooks, composition, prop drilling)
- Accessibility (ARIA attributes, keyboard navigation, screen reader support)
- Performance (unnecessary re-renders, bundle size, lazy loading)
- CSS/styling best practices
- Client-side security (XSS prevention, CSP compliance)

Output findings using severity levels: 🔴 Bug, 🟡 Concern, 🟢 Suggestion.
```

### Example: DevOps Automation Plugin

```text
devops-toolkit/
├── plugin.json
├── agents/
│   └── infra-planner.agent.md
├── skills/
│   ├── terraform-plan/
│   │   ├── SKILL.md
│   │   └── validate-plan.sh
│   └── k8s-debug/
│       └── SKILL.md
├── hooks.json
└── .mcp.json
```

**plugin.json:**
```json
{
  "name": "devops-toolkit",
  "description": "Infrastructure planning, Terraform management, and Kubernetes debugging",
  "version": "1.0.0",
  "author": { "name": "Platform Team" },
  "license": "MIT",
  "keywords": ["devops", "terraform", "kubernetes", "infrastructure"],
  "hooks": "hooks.json",
  "mcpServers": ".mcp.json"
}
```

**.mcp.json:**
```json
{
  "mcpServers": {
    "kubectl": {
      "type": "local",
      "command": "npx",
      "args": ["@modelcontextprotocol/server-kubernetes"],
      "tools": ["*"]
    }
  }
}
```

**hooks.json:**
```json
{
  "version": 1,
  "hooks": {
    "preToolUse": [
      {
        "type": "command",
        "bash": "#!/bin/bash\nINPUT=$(cat)\nTOOL=$(echo \"$INPUT\" | jq -r '.toolName')\nARGS=$(echo \"$INPUT\" | jq -r '.toolArgs')\nif [ \"$TOOL\" = \"bash\" ] && echo \"$ARGS\" | grep -qE 'kubectl delete|terraform destroy'; then\n  echo '{\"permissionDecision\":\"deny\",\"permissionDecisionReason\":\"Destructive infra commands require manual approval\"}'\nfi",
        "comment": "Block destructive infrastructure commands"
      }
    ]
  }
}
```

## Rules When Creating Plugins

1. Plugin names must be kebab-case, max 64 characters
2. Always include a meaningful `description` in plugin.json — this is what users see when browsing
3. Agent files must use the `.agent.md` extension
4. Skill directories must contain a `SKILL.md` file (exact name, case-sensitive)
5. Hook scripts must be executable (`chmod +x`) and have proper shebangs (`#!/bin/bash`)
6. Hook scripts read JSON from stdin and (for preToolUse) write JSON to stdout
7. Use `jq` for JSON parsing in hook scripts — it is widely available
8. MCP server `type: "stdio"` maps to `type: "local"` — both work
9. All paths in plugin.json are relative to the plugin root directory
10. Plugins cannot override built-in tools or agents
11. Plugins cannot override project-level or user-level agents/skills (first-found-wins)
12. Version your plugin with semver for marketplace distribution
13. Make hook scripts cross-platform when possible (provide both `bash` and `powershell`)
14. Keep agent prompts under 30,000 characters
15. Test locally with `copilot plugin install ./path` before distributing
